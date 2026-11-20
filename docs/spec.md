# rpkit specification

Status: draft · Version: 0.1

rpkit is a self-hostable relying-party (RP) implementation for [World ID](https://docs.world.org/world-id). It lets an application request World ID proofs, verify them and pay protocol fees, with no hosted service between the RP and the protocol.

This document describes what rpkit does, how its parts fit together, and which guarantees it gives. It is written for engineers who integrate rpkit, operate it, or contribute to it.

## 1. Motivation

Today a World ID RP depends on two pieces of TFH-run software:

- **IDKit** builds requests in the RP's frontend, shows the QR code and polls the bridge for the answer.
- **The Developer Portal** registers the RP on-chain, verifies proofs and serves the RP's name and logo to World App.

That has four costs:

- **Dependency.** RPs must trust and depend on a hosted service for a decision that the protocol could make on its own.
- **Privacy.** The hosted service sees every verification: which RP, which action, when, the nullifiers and the signals. That gives it a usage record for every RP and every user interaction, which the protocol itself never needs.
- **Gaps.** Important guarantees fall between components and nobody enforces them: a nonce is used once, a nullifier is used once.
- **Speed.** Each new proof type needs work in several places before RPs can use it.

rpkit moves everything an RP needs into one open-source stack the RP runs itself. The protocol's on-chain contracts remain the source of truth.

## 2. Goals and non-goals

### Goals

- **Self-hostable.** An RP can run every rpkit component on its own infrastructure, using only a World Chain RPC endpoint and the World ID protocol.
- **Private.** No third party learns about a verification beyond what the protocol requires (§11.1).
- **Every proof type.** rpkit verifies all World ID proof types, and adding a new one is a contained change.
- **Safe by default.** Nonce and nullifier uniqueness are built in, not left to the integrator. Unknown inputs are rejected, and every component fails closed.
- **Language-neutral.** RP backends in any language integrate through HTTP and standard JWTs. Rust services can link the library directly.
- **Operable.** Health probes, metrics, bounded timeouts and documented failure modes come with the first release.

### Non-goals

- rpkit is not a wallet or authenticator, and it never generates proofs.
- rpkit does not replace protocol infrastructure such as OPRF nodes, the indexer or the contracts.
- rpkit does not issue credentials.

## 3. Terminology

| Term | Meaning |
|---|---|
| RP | Relying party: the application that asks a user for a World ID proof. |
| `rp_id` | The RP's identifier in the on-chain `RpRegistry`. |
| Signer key | The key registered for the RP; it signs every proof request. |
| Manager key | The key allowed to update the RP's registry entry (signer, domain, status). |
| Proof request | A `ProofRequest` from [`world-id-primitives`](https://github.com/worldcoin/world-id-protocol/tree/main/crates/primitives): what the RP asks for, signed by the RP. |
| Proof response | A `ProofResponse`: what the authenticator returns. |
| Proof type | The `proof_type` of a request: `uniqueness`, `session` or `embedding_similarity` today. |
| Nonce | A random value the RP puts in each request. It MUST be accepted at most once. |
| Nullifier | In a `uniqueness` proof, a value derived from the user and the RP's action: the same person always produces the same nullifier for the same action. In a `session` proof, a one-time value bound to a random per-proof action. |
| Session id | A stable, RP-scoped identifier for a user's session. `session` proofs show that the same person is still behind it. |
| Bridge | The encrypted relay that carries requests and responses between the RP's frontend and the user's authenticator. |
| Transaction | One proof request and its lifecycle inside rpkit (§7). |

The key words MUST, SHOULD and MAY are used as in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## 4. Overview

rpkit has four parts. Only one of them is a service.

| Part | Kind | Responsibility |
|---|---|---|
| `rpkit-core` | Rust library | Builds and signs requests, verifies proofs of every type, defines the storage and signing interfaces. |
| `rpkit-server` | Service (container image, Helm chart) | Runs `rpkit-core` behind an HTTP API. It owns transactions, nonce and nullifier state, keys, registration and billing. |
| `@rpkit/widget` | Frontend component | Shows the pairing QR code or deep link and the transaction status. It holds no secrets and does no cryptography. |
| Edge helpers | Small libraries (TypeScript, Go, Python) | Verify the signed result JWT and webhook signatures that `rpkit-server` emits. |

```mermaid
flowchart LR
  subgraph RP["Relying party infrastructure"]
    W["@rpkit/widget"]
    B["RP backend"]
    S["rpkit-server"]
    DB[("Postgres or DynamoDB")]
    K["KMS"]
  end
  subgraph P["World ID protocol"]
    BR["Bridge"]
    A["Authenticator (World App)"]
    C["World Chain contracts"]
  end
  W -->|start| B -->|create transaction| S
  S --> DB
  S --> K
  S -->|eth_call, register, billing| C
  S <-->|WIP-110 encrypted channel| BR
  W -.->|status| S
  A <--> BR
```

There are two integration modes:

- **Service mode (default).** The RP runs `rpkit-server` next to its backend, as a sidecar or a shared service. The backend creates transactions and receives a signed result. This is the recommended mode for all languages.
- **Library mode.** A Rust backend links `rpkit-core` directly and provides its own `NonceStore` and `NullifierStore` implementations. Those implementations MUST pass the conformance tests in `rpkit-core`.

## 5. Proof verification

### 5.1 Verifier registry

`rpkit-core` holds one verifier per proof type, keyed by `proof_type`:

```rust
#[async_trait]
pub trait ProofVerifier: Send + Sync {
    /// The proof type this verifier handles.
    fn proof_type(&self) -> ProofTypeId;

    /// Checks `response` against the RP's own `request`.
    async fn verify(
        &self,
        request: &ProofRequest,
        response: &ProofResponse,
    ) -> Result<Verified, VerifyError>;
}
```

The steps around the verifier are the same for every type: request matching, nonce consumption and nullifier recording. A verifier only implements what is specific to its proof type.

A request with a proof type that has no registered verifier MUST be rejected.

### 5.2 Steps for every proof type

For each response, rpkit runs these steps in order and stops at the first failure:

1. **Match.** The response answers this request: same request `id`, and every response item matches a request item and satisfies the request's constraints (`ProofRequest::validate_response`).
2. **Freshness.** The request has not expired (`expires_at`).
3. **Type-specific verification.** The verifier for the request's `proof_type` checks the proof (§5.3).
4. **Nonce.** The request's nonce is consumed atomically (§7.2). A nonce that was already consumed rejects the proof.
5. **Nullifier.** If the proof type produces nullifiers, each one is recorded, unique (§7.3). A nullifier already recorded rejects the proof as `nullifier_reused`.

The nonce is consumed after the proof check, so an invalid proof cannot burn a legitimate request.

What the nullifier step means depends on the proof type:

| Proof type | Nullifier | What uniqueness protects against |
|---|---|---|
| `uniqueness` | Per user and action | The same person acting twice under one action |
| `session` | Session nullifier, one-time | Replay of a session proof; the stable user handle is the `session_id` |
| `embedding_similarity` | None | Not applicable; replay is covered by the nonce |

### 5.3 Proof types

| Proof type | Where it is verified | What the verifier checks |
|---|---|---|
| `uniqueness` | On-chain, through `WorldIDVerifier.verify` (or `verifyProofAndSignals` when bound to a session) with `eth_call` | The Merkle root is valid; the issuer key is registered for the credential schema; the OPRF key for `rp_id` exists; the credential has not expired; the Groth16 proof verifies over the public signals, including `rp_id`, `action`, `nonce` and `signal_hash` |
| `session` | On-chain, through `WorldIDVerifier.verifySession` with `eth_call` | As above, with the session id and session nullifier instead of an action |
| `embedding_similarity` | Off-chain, through [`world-id-rp`](https://github.com/worldcoin/world-id-protocol/tree/main/crates/rp) (WIP-202) | The Flamingo Verifier's Nitro attestation and its key; accepted engine configurations; proof recency; Merkle root and issuer key, both checked on-chain; the AAT policy; the proof itself |

`eth_call` is free and touches no state, so on-chain verification costs the RP nothing beyond RPC usage.

For `embedding_similarity`, the RP stays responsible for one thing rpkit cannot check: the challenge image behind the request's `challenge_hash` must be the image the RP checked against the person it is dealing with. rpkit takes `challenge_hash` as an input when the transaction is created.

Some credentials need checks beyond the proof itself. Integrity bundles for selfie credentials are one example. These run as **credential plugins** that the verifier calls for the matching `issuer_schema_id`.

### 5.4 Adding a proof type

A new proof type needs:

1. a `ProofVerifier` implementation in `rpkit-core`
2. a policy section in the configuration, if the type has RP-chosen parameters
3. test vectors.

The HTTP API, the stores, the result format and billing do not change.

### 5.5 Policy

Some proof types depend on RP choices. These live in a versioned configuration file (TOML), validated at startup. An invalid policy prevents the server from becoming ready.

```toml
[embedding_similarity]
issuer_version = 1
engine_config_hashes = ["0x…"]

[[embedding_similarity.nitro_pcrs]]
pcr0 = "…"
pcr1 = "…"
pcr2 = "…"

[embedding_similarity.aat]
required = true
trust_anchor_key_hashes = ["0x…"]
accepted_flags = [{ platform = 1, sec_level = 2, sec_meta = 0, authenticator_meta = 0 }]
```

Empty allowlists are rejected. rpkit has no "accept anything" default.

## 6. Requests and keys

### 6.1 Request signing

Every proof request carries the RP's signature. Authenticators and OPRF nodes check it against the signer key registered for `rp_id`.

rpkit produces the same signature as IDKit's `signRequest`:

- **Scheme:** EIP-191 `personal_sign` over secp256k1.
- **Message:** `0x01 ‖ nonce (32 bytes) ‖ created_at (u64, big-endian) ‖ expires_at (u64, big-endian) ‖ action (32 bytes, if the proof type has one)`.
- **Nonce:** 32 random bytes, reduced into the BN254 scalar field.

The default request lifetime is 5 minutes. The OPRF nodes accept `created_at` within ±5 minutes of their clock and a lifetime of at most 30 minutes.

### 6.2 Signer interface

```rust
#[async_trait]
pub trait Signer: Send + Sync {
    /// The Ethereum address of the signer key.
    fn address(&self) -> Address;
    /// Signs a 32-byte EIP-191 digest.
    async fn sign_digest(&self, digest: B256) -> Result<Signature, SignerError>;
}
```

Implementations:

- `LocalSigner`: an in-process key, for development and tests only. The server refuses it unless dev mode is set.
- `AwsKmsSigner`: an AWS KMS `ECC_SECG_P256K1` key. This is the production default.
- Further backends (GCP KMS, HashiCorp Vault) implement the same trait.

Keys:

- The **manager key** MUST be a different key from the signer key. It is used only for registry updates.
- **Rotation:** register the new signer with `RpRegistry.updateRp`, signed by the manager key; then switch the server to the new key.
- **One signer per RP.** The signed message does not include `rp_id` (§11), so a signer shared between RPs would let one RP's requests be replayed as another's. rpkit MUST refuse a configuration in which two RPs share a signer address.

## 7. State

### 7.1 Transactions

In service mode each proof request is a transaction:

| Field | Description |
|---|---|
| `tx_id` | Random identifier. |
| `rp_id`, `action`, `proof_type` | What is being asked. |
| `request` | The signed `ProofRequest`. |
| `nonce` | The request's nonce. |
| `bridge_session` | The bridge host, session id, seat token, pairing secret and handshake keys. Deleted when the transaction ends. |
| `status_token` | A random token that lets the widget read this transaction's status, and nothing else. |
| `status` | `created` → `paired` → `completed`, or `failed` or `expired`. |
| `result` | The verified result or the failure class. |
| `expires_at` | Equal to the request's `expires_at`. |

Expired transactions are deleted by a background sweep.

### 7.2 Nonces

```rust
#[async_trait]
pub trait NonceStore: Send + Sync {
    /// Atomically marks `nonce` used for `rp_id`. Returns `AlreadyUsed` if it was.
    async fn consume(&self, rp_id: RpId, nonce: FieldElement, expires_at: u64)
        -> Result<Consumed, StoreError>;
}
```

`consume` MUST be atomic across all replicas: a compare-and-set, a unique insert or a conditional write. A store error is reported as `unavailable`, never as success.

### 7.3 Nullifiers

```rust
#[async_trait]
pub trait NullifierStore: Send + Sync {
    /// Records `nullifier`. Returns `AlreadyRecorded` if it exists for `(rp_id, scope)`.
    async fn record(&self, rp_id: RpId, scope: NullifierScope, nullifier: FieldElement)
        -> Result<Recorded, StoreError>;
}
```

`NullifierScope` is the action for `uniqueness` proofs and the session id for `session` proofs.

Retention:
- `uniqueness` nullifiers are kept for as long as the action is in use.
- Session nullifiers can be dropped once the request that carried them has expired.

A reused nullifier is always a rejection: `nullifier_reused` is a terminal error, never a success with a warning.

### 7.4 Backends

| Backend | Use |
|---|---|
| In-memory | Development and tests. The server refuses it outside dev mode. |
| Postgres | Default for production. Migrations run as a separate `rpkit migrate` command, not at startup, so rolling deploys and rollbacks stay safe. |
| DynamoDB | AWS-native option, using conditional writes. |

Every backend passes the same conformance test suite.

## 8. Trust and RP identity

**Who is asking.** The on-chain `RpRegistry` binds `rp_id` to a signer key, a manager key and a domain. Registration is permissionless and pays a registration fee.

At startup, `rpkit-server` MAY register the configured RP itself. Registration is idempotent: rpkit checks the registry first and only registers if the entry is missing. After registration, the protocol generates the RP's OPRF key asynchronously. Until that key exists, proofs fail, so rpkit reports that RP as degraded (§13) and rejects its transactions.

**What the user sees.** The consent screen in World App shows the RP's name, logo and action description. Today World App reads these from the Developer Portal. The protocol is moving to a domain-verified metadata record:

- published under `/.well-known/` on the domain registered in `RpRegistry`
- committed by hash on-chain.

rpkit will serve and publish that record once the format is specified. Until then, an RP still needs a Developer Portal entry for its consent metadata (§12).

## 9. Transport

rpkit carries requests and responses over the World ID Message Bridge, as specified in [WIP-110](https://github.com/worldcoin/world-id-protocol/pull/1117) (draft). WIP-110 is a generic end-to-end encrypted channel between two parties over an untrusted relay:

- **Pairing:** the party that creates the session (the **Initiator**) shows a pairing link containing a random secret `s` and the session's bridge host `b`. The other party (the **Responder**) opens it and joins.
- **Handshake:** the two parties run a three-message handshake combining X-Wing (post-quantum) key exchange with a key derived from `s`.
- **Records:** application messages then travel as ChaCha20-Poly1305 encrypted records.

The bridge sees only metadata and cannot read or forge messages.

### 9.1 Roles

**rpkit-server is the Initiator; the authenticator (World App) is the Responder.** The browser never takes part in the channel. The widget only shows the pairing link and the transaction status.

Putting the Initiator on the server has three effects:

- **The proof goes straight to rpkit.** It never passes through the browser, so a tampered page cannot fake, swap or withhold a result.
- **No secrets or cryptography in the frontend.** The widget is a display component.
- **No browser needed.** RPs without a browser, such as a backend service, a kiosk or a call bot, use the same flow: they show the pairing link however they can (QR code, deep link, message).

The cost: rpkit holds one long-poll per open transaction (WIP-110 caps a poll at 30 seconds and a session at one hour). The server limits the number of open transactions and sheds new ones with `503` when it is full.

### 9.2 Flow

1. The RP backend creates a transaction (`POST /v1/tx`).
2. rpkit mints the nonce and signs the `ProofRequest` (§6). It then generates the WIP-110 secrets: the pairing secret `s`, its seat token and an X-Wing key pair.
3. rpkit creates a bridge session, sends handshake message 1, and returns the pairing URI to the RP backend.
4. The widget shows the pairing URI as a QR code, or as a deep link on mobile.
5. The authenticator opens the URI, joins the session and completes the handshake.
6. rpkit sends the signed `ProofRequest` as a record.
7. The authenticator shows the consent screen and generates the proof. It sends the `ProofResponse` as a record, or an error record if the user declines.
8. rpkit verifies the response (§5), leaves the session and deletes all session secrets.
9. The RP backend fetches the signed result (`GET /v1/tx/{id}`). The widget learns the outcome from the status stream.

```mermaid
sequenceDiagram
  participant U as Browser + widget
  participant B as RP backend
  participant S as rpkit-server (Initiator)
  participant BR as Bridge
  participant A as Authenticator (Responder)
  participant C as World Chain
  U->>B: start
  B->>S: POST /v1/tx
  S->>S: mint nonce, sign request, generate s, seat token, X-Wing keys
  S->>BR: create session, handshake message 1
  S-->>B: tx_id, pairing URI, status token
  B-->>U: pairing URI, status token
  U->>U: show QR code, subscribe to status
  A->>BR: join with s, handshake messages 2 and 3
  S->>BR: record: signed ProofRequest
  A->>A: consent, generate proof
  A->>BR: record: ProofResponse
  BR-->>S: ProofResponse
  S->>C: chain checks (eth_call)
  S->>S: type checks, consume nonce, record nullifier
  S->>BR: leave session
  S-->>U: status: completed
  B->>S: GET /v1/tx/{id}
  S-->>B: signed result JWT
```

### 9.3 World ID proof profile

WIP-110 leaves several choices to each application. The World ID proof request profile that fixes them is not yet specified. Until it is, rpkit uses the values below and treats them as provisional:

| Choice | Provisional value |
|---|---|
| Pairing URI | `https://world.org/verify#s=…&b=…&d=…`. Parameters go in the fragment so they never reach a web server. A `worldid://` deep link carries the same parameters. |
| Authentication mode | `psk`: possession of the pairing URI authenticates the authenticator. `code` mode (a 6-letter code typed into World App) is a per-RP option for high-value flows. |
| `app_label` | `worldid/rp/v1/proof` |
| `app_context` | `d`: SHA-256 of the signed `ProofRequest`, also carried in the URI. The authenticator rejects a request record whose digest differs, so a malicious bridge cannot substitute a request even before the handshake completes. |
| Records | Initiator: one `ProofRequest`. Responder: one `ProofResponse` or one error. Either side leaves after that. |
| Session `ttl` | The request lifetime (§6.1), so the channel and the request expire together. |

rpkit will follow the profile once it is published and expose the WIP-110 handshake version in its metrics.

### 9.4 Binding the result to the right user

The RP backend MUST link each `tx_id` to the user session that created it, and accept a result only within that session.

One attack remains that no cross-device flow can fully prevent. An attacker starts a transaction in their own session and gets a victim to scan the attacker's QR code, which attaches the victim's proof to the attacker's account. This is the cross-device phishing described in the [IETF cross-device security BCP](https://datatracker.ietf.org/doc/draft-ietf-oauth-cross-device-security/).

Mitigations:

- World App's consent screen shows the RP's verified domain (§8).
- Transactions are short-lived.
- Same-device flows (§9.5) are preferred where the user is already on their phone.

### 9.5 Same device

- **Mobile browser:** the widget opens the pairing URI as a deep link. World App joins the session exactly as in §9.2 and returns the user to the page afterwards (`return_to`).
- **Inside World App:** a page running inside World App passes the signed request to the app through its native interface and receives the `ProofResponse` back. The page hands the response to the RP backend, which submits it with `POST /v1/tx/{id}/response`. Verification is the same as in §9.2 steps 8–9.

### 9.6 Bridge hosts

WIP-110 requires the authenticator to check the bridge host `b` against an allowlist. An RP can run its own bridge only once authenticators accept its host. Until then, rpkit uses the public World ID bridge.

The bridge's errors map onto rpkit's error classes:

| Bridge error | rpkit's response |
|---|---|
| `rate_limited`, `unavailable` | Retry with backoff, honoring `Retry-After`, within the transaction's deadline |
| `seat_taken` | The pairing link was claimed by another device: the transaction fails with `pairing_hijacked` and the widget tells the user |
| `not_found`, `forbidden`, `conflict` | The transaction fails |

### 9.7 Legacy bridge

World App today speaks the earlier bridge protocol:

- a single AES-256-GCM key sent in the link's query string
- `POST /request`, then polling `GET /response/{id}`.

Until World App supports WIP-110 for proof requests, rpkit also implements the legacy bridge, using the same server-side Initiator model: rpkit generates the key, posts the request and polls for the response. The RP API and the widget are the same for both transports; the transport is chosen in configuration. Legacy support will be removed once WIP-110 is deployed in World App.

### 9.8 Later: OpenID4VP and the Digital Credentials API

A later version adds an OpenID4VP profile over the browser [Digital Credentials API](https://www.w3.org/TR/digital-credentials/), with the registry acting as the verifier's trust anchor. The bridge stays the default until browsers and authenticators support that profile.

## 10. API

`rpkit-server` listens on two ports: the API port and an admin port that is never exposed publicly.

### 10.1 RP API

Authenticated. In sidecar deployments it is bound to localhost; shared deployments add bearer-token or mTLS authentication. All paths are versioned under `/v1`.

| Method and path | Purpose |
|---|---|
| `POST /v1/tx` | Create a transaction for `{ rp_id, proof_type, action, signal, credentials, constraints }`. Returns `{ tx_id, pairing_uri, status_token, expires_at }`. |
| `GET /v1/tx/{id}` | Transaction status and, once completed, the signed result JWT. |
| `GET /v1/tx/{id}/events` | Server-sent events for status changes. A late subscriber receives the current state first. |
| `POST /v1/tx/{id}/response` | Submit a `ProofResponse` received outside the bridge, as in the in-app flow (§9.5). |
| `POST /v1/verify` | Stateless verification of `{ request, response }` for RPs that keep their own nonce and nullifier state. Runs §5.2 steps 1–3 only. |

### 10.2 Public API

The only public endpoint is `GET /v1/public/tx/{id}/events?token=<status_token>`.

- It streams status changes as server-sent events: `created`, `paired`, `completed`, `failed`, `expired`.
- It never returns the result or any proof data.
- The RP exposes it to browsers directly or proxies it through its backend; the RP API stays private either way.

Later versions add the OpenID4VP `request_uri` and `direct_post` endpoints here.

### 10.3 Results

A completed transaction's result is a JWT signed with ES256:

- **Claims:** `iss`, `aud` (the RP), `iat`, `exp`, `tx_id`, `rp_id`, `proof_type`, `action` where present, `session_id` where present, and per response item `issuer_schema_id` and `nullifier` where the proof type has one.
- **Keys:** published at `/.well-known/jwks.json`. A new key is added before use, the newest key signs, and an old key is retired after the longest token lifetime has passed.
- **Webhooks:** optional. Delivered as signed webhooks (id, timestamp and HMAC). Receivers must deduplicate on the id.

### 10.4 Errors

Errors have a stable `code` and a class:

| Class | HTTP | Meaning | Retry? |
|---|---|---|---|
| `invalid` | 422 | The proof or request is rejected: `proof_invalid`, `request_expired`, `nonce_reused`, `nullifier_reused`, `policy_rejected`, `unknown_proof_type`. | No |
| `rp_degraded` | 503 | The RP cannot take proofs right now: `oprf_key_pending`, `billing_capacity_exhausted`. | After the cause is fixed |
| `pending` | 409 | The transaction is not finished yet. | Yes |
| `unavailable` | 503 | A dependency failed: RPC, store or KMS. | Yes, with backoff |
| `bad_request` | 400 | Malformed input. | No |

## 11. Security and privacy considerations

### 11.1 Privacy

What each outside party learns from a verification:

| Party | What it sees |
|---|---|
| TFH / Developer Portal | Nothing on the verification path. The portal only serves consent metadata until the domain-verified record is supported (§15). |
| World Chain RPC provider | The `eth_call` inputs: `rp_id`, nullifier, action, nonce, signal hash and timing. Run your own World Chain node, or use a provider you trust with this. |
| Bridge operator | Session ids, message sizes, timing and IP addresses, of the rpkit server and of the user's device (WIP-110 §4). |

rpkit sends no telemetry to anyone. Its metrics and logs never contain nullifiers, signals or session ids.

### 11.2 Security

- **Fail closed.** Unknown proof types, empty policies, unreachable dependencies and store errors all reject. The default configuration exposes nothing publicly and enables no development backends.
- **Replay.**
  - Nonces are single-use (§7.2) and requests are short-lived.
  - The protocol itself does not enforce single-use nonces globally: `WorldIDVerifier` is a view function, and OPRF nodes only filter replays in memory. rpkit's nonce store is the enforcement point.
- **Duplicate humans.** Nullifiers are unique per `(rp_id, action)` (§7.3).
- **Cross-RP replay.**
  - The signed request message does not cover `rp_id`. rpkit refuses shared signers (§6.2).
  - A protocol change adding `rp_id` and the chain id to the signed message (message version `0x02`) is proposed. rpkit will support both versions during the migration.
- **Trust in RPC.** rpkit trusts its World Chain RPC endpoint to answer `eth_call` honestly. Production deployments SHOULD use an RPC they operate or trust, and MAY cross-check across several providers. Cached chain values (valid roots, issuer keys) have short lifetimes, and negative results are never cached.
- **Keys.** Production keys live in a KMS. The signer and manager keys are separate.
- **Inputs.**
  - Request bodies are size-limited and parsed strictly, rejecting unknown fields where the protocol types allow it.
  - Webhook targets are checked against an allowlist and cannot point at private or loopback addresses.

## 12. Fees

World ID charges per-proof fees through the `WorldIDBilling` contract (WIP-107):

- An RP buys monthly capacity in WLD, priced in USD per World ID, with `purchaseCapacity(scope, amount, maxWld)`.
- Billing applies only to RPs flagged for billing, and today only to `embedding_similarity` proofs.
- The user's authenticator registers against the RP's capacity, so the fee is enforced by the protocol, not by rpkit.

rpkit's billing client:

- tops up capacity with a `maxWld` slippage limit
- exposes remaining capacity as a metric
- marks a billed RP as degraded when its capacity is exhausted: transactions for that RP fail fast with `billing_capacity_exhausted`, and the other RPs on the server are unaffected.

A fork of rpkit cannot skip fees, so rpkit needs no license enforcement and stays fully open source.

## 13. Operations

- **Health** (admin port):
  - `/health`: the process is alive.
  - `/ready`: policy loaded; store, RPC and KMS reachable. Any of these failing makes the server not ready.
  - **Per-RP status**, also on the admin port: OPRF key present and billing capacity available. A degraded RP does not affect other RPs on the same server, and its degraded state is exposed as a metric for alerting.
- **Timeouts and retries.** Every remote call has a deadline. Only idempotent calls (`eth_call`, reads) are retried, at most twice, with exponential backoff and jitter. Inbound requests have a concurrency limit and are shed with 503 under overload.
- **Metrics** (DogStatsD/OTLP):
  - verifications by proof type and outcome
  - nonce and nullifier conflicts
  - RPC latency and errors by call
  - cache hits
  - transaction lifecycle
  - billing capacity
- **Logs.** Structured JSON with trace id, `rp_id`, dependency, upstream status and failure class. Failures are logged at warn or error; successes are counted in metrics, not logged.
- **Deployment.**
  - Container images on GitHub Container Registry, signed with build provenance.
  - A first-party Helm chart.
  - Configuration as TOML plus environment variables.

## 14. Versioning

These are versioned independently:

| What | How |
|---|---|
| HTTP API | Path prefix (`/v1`). Breaking changes add a new prefix; the old one stays for at least one minor release. |
| Proof request format | The `version` field of `ProofRequest`, as defined by the protocol. |
| Request signature | The leading message byte (`0x01`, later `0x02`). |
| rpkit releases | Semantic versioning. A release's database schema is compatible with the previous minor release, so rollbacks are safe. |

## 15. Known limitations

- **Bridge.**
  - World App speaks only the legacy bridge today, so rpkit supports it until WIP-110 is deployed (§9.7).
  - Authenticators accept only allowlisted bridge hosts, so RPs use the public bridge (§9.6).
  - The World ID proof profile for WIP-110 is not yet specified (§9.3).
- **Consent metadata.** World App currently reads the RP's name and logo from the Developer Portal. Until the domain-verified record (§8) is supported, an RP needs a Developer Portal entry for display purposes, even though verification no longer goes through it.
- **Legacy proofs.** World ID v3 proofs can only be verified by TFH's sequencer, so rpkit does not support them.

## 16. Roadmap

| Phase | Scope |
|---|---|
| 0 | `rpkit-core` with the `uniqueness`, `session` and `embedding_similarity` verifiers; stateless `POST /v1/verify`; container image. |
| 1 | Transactions, WIP-110 and legacy bridge transports, nonce and nullifier stores, KMS signer, result JWT, registration, billing, Helm chart. |
| 2 | `@rpkit/widget`, edge helpers, domain-verified metadata, request signature `0x02`. |
| 3 | OpenID4VP / Digital Credentials API profile, conformance test vectors, external audit. |
