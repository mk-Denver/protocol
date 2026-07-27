# PIP-01: Escrow Descriptor

## Status

- Status: Draft
- Implementation: Required
- Scope: public escrow declaration for swap compatibility and execution assumptions
- Related:
  - [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
  - [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md)

## Purpose

This document defines the public escrow descriptor event referenced by agent definitions and swaps.

## Event Type

- kind: `30361`
- addressable
- `d` tag: stable identifier for one escrow configuration

## Function

The escrow descriptor tells counterparties and operators:

- what escrow mechanism is used
- which settlement or invoice networks are supported
- what the funding and release rules are
- how the escrow instance is referenced
- what timeout and dispute assumptions apply
- whether and how the escrow may be invoked as a standalone service

## Minimum Content

`content` is JSON and MUST be versioned.

Minimum expected fields:

- `version`
- `escrow_type`
- `networks`
- `funding_rules`
- `release_rules`
- `dispute_rules`
- `reference_format`
- `updated_at`

## Standalone Sufficiency

An escrow descriptor has two intended use levels:

- **compatibility and discovery** — the descriptor declares which networks, assets, reference formats, funding rules, release rules, and dispute policy an escrow supports, so that agents and swaps can select it
- **standalone operation** — the descriptor additionally declares the public service interface required to instantiate and operate an escrow instance without out-of-band negotiation

A descriptor that omits the `service` block (see Service Interface) is sufficient for compatibility and discovery and for use inside Pontmore swap flows where the swap state machine in [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md) carries execution. It is NOT sufficient for a standalone application to create, fund, observe, release, or refund an escrow instance on its own.

A descriptor that includes a `service` block and the required `schema_url` is intended to be sufficient for standalone application use. When `service` is present, the descriptor is the primary source of truth for the public escrow service interface.

The minimum content fields in Minimum Content remain required in both use levels. The `service` block is the additional, optional field that upgrades a descriptor from discovery-only to standalone-sufficient.

## Service Interface

The `service` content field advertises the public interface that a standalone application uses to instantiate and operate an escrow instance. It is OPTIONAL at the descriptor level. When present, it MUST conform to this section. A standalone client MUST reject a descriptor whose `service` block is incomplete, whose transport or authentication choices are unsupported or untrusted, or whose `schema_url` does not normatively define the advertised wire contract.

### Service Fields

When `service` is present, it MUST include:

- `transport`
  - non-empty array of supported service transports
  - canonical first transport is `https`
- `interface`
  - interface and version identifier; example: `pontmore_escrow_http_v1`
- `endpoint`
  - canonical endpoint for the first advertised transport
  - when `https` is advertised, the endpoint MUST be an absolute `https://` URL
  - each additional advertised transport MUST resolve to exactly one endpoint using that transport's required scheme, as defined by `schema_url`
- `auth`
  - non-empty array of supported authentication methods; see Authentication
- `operations`
  - MUST include `create`, `funding_instructions`, `fund_status`, `release`, `refund`, and `cancel` for standalone service use
- `funding_model`
  - multi-party funding model; see Funding Model
- `release_decisions`
  - non-empty array of accepted release-decision formats; see Release Decisions
- `schema_url`
  - mandatory machine-readable schema URL for the normative wire contract; see Wire Contract
  - MUST use `https://`
  - MUST resolve only to an immutable or versioned schema artifact, preferably with a digest or immutable URL
  - MUST NOT point to a private, loopback, link-local, multicast, or otherwise disallowed network destination

### Authentication

For HTTPS transports, the recommended authentication method is `nostr_http_auth`, defined as NIP-98 (Nostr HTTP Authentication).

- a participant authenticates by signing each HTTP request with its Nostr key, producing a NIP-98 authorization header
- identity is the participant's Nostr pubkey; the descriptor and escrow operator MUST NOT require published bearer secrets for public protocol operations
- `create` uses an explicitly authorized incremental enrollment flow: the initial authenticated request establishes the escrow instance, and any later participant join MUST use a service-issued invitation or enrollment token plus the joining participant's authenticated request
- the idempotency key is for request deduplication only and MUST NOT authorize participant joining or bind any additional pubkeys
- the operator binds subsequent `fund_status`, `release`, `refund`, and `cancel` operations to the pubkeys that have been explicitly enrolled for that escrow instance
- each participant signs its own operations; threshold release decisions require signatures from the declared threshold of bound participants
- operators MAY require additional private operator-layer authorization for back-office actions, but such authorization is an operator overlay and MUST NOT be advertised as a public descriptor field
- authentication identifiers MUST be registered methods; `nostr_http_auth` is the canonical HTTP method, and any extension method MUST be defined by `schema_url` with a mandatory interface schema and versioned identifier

### Wire Contract

`schema_url` MUST point to a schema that normatively defines the public service contract, either by embedding the contract itself or by referencing a complete protocol-native or OpenAPI schema. That schema MUST define:

- HTTP methods and paths for `create`, `fund_status`, `release`, `refund`, `cancel`, and funding-instruction retrieval
- request and response schemas for each listed operation
- error formats and HTTP status codes
- escrow state transitions and terminal states
- idempotency rules, including the shared correlation value used by repeated `create` requests
- how `funding_model` and `release_decisions` values map to wire payload fields and validation rules
- transport-specific endpoint resolution rules for every advertised transport
- schema fetch requirements, including HTTPS-only retrieval, redirect limits, bounded response size, allowed content types, and rejection of private or otherwise disallowed network destinations

### Operations

The canonical escrow instance operation vocabulary is:

- `create` — open a new escrow instance and bind its participant pubkeys. The initial authenticated request establishes the escrow instance. Any later participant join MUST use a service-issued invitation or enrollment token plus the joining participant's authenticated request. Repeated calls with the same idempotency key are idempotent and MUST NOT create a second escrow instance, but the idempotency key MUST NOT authorize participant joining or bind any additional pubkeys.
- `funding_instructions` — retrieve the funding instructions needed to fund the escrow instance
- `fund_status` — observe the funding state of a participant's side
- `release` — request release of the escrowed amount to the winner or payee
- `refund` — request refund of the escrowed amount to its funder
- `cancel` — request cancellation of an unfunded or unresolved escrow instance

Example shared-instance create flow:

```text
participant A -> create()
service -> invitation=enroll-abc123
participant B -> create(invitation=enroll-abc123)
```

The shared invitation or enrollment token is the authorization value for participant joining. The idempotency key, if present, is only the correlation value used to deduplicate retries.

Operations not in this vocabulary MAY be advertised but are non-canonical and MUST be documented by the operator's referenced schema.

### Funding Model

The `funding_model` field declares how many participants fund a single escrow instance:

- `single_funder` — one participant funds the escrow
- `two_party` — two participants each fund a side of the same escrow instance
- `n_of_m` — the referenced schema MUST expose two normative fields: `funding_threshold` and `participant_count`. `funding_threshold` is `M`, the minimum number of required funders. `participant_count` is `N`, the total declared participants. Clients MUST use these values consistently when validating funding and MUST NOT treat the escrow as active until both values are known and satisfy the declared funding condition.

### Release Decisions

`release_decisions` advertises the generic decision formats the operator accepts to authorize a `release` or `refund`. The minimum canonical vocabulary is:

- `mutual_consent` — release or refund authorized by all declared participants
- `operator_decision` — release or refund authorized by the escrow operator as arbiter
- `oracle_signature` — release or refund authorized by a signature from a referenced oracle
- `application_signed_result` — release or refund authorized by a signed result from the originating application
- `threshold_participant_signatures` — release or refund authorized by a threshold of participant signatures

Each release-decision format MUST be defined by the referenced schema with a verifiable standalone contract:

- `application_signed_result`
  - the schema MUST define the signed payload, the signer identity, the signature encoding, the escrow binding, the result binding, and replay protection
  - the signed payload MUST include a stable escrow identifier and a result identifier or hash so the result cannot be replayed against another escrow instance
  - the signer identity MUST be a named application identity that the schema or service advertises as valid for this decision type
  - signatures MUST be serializable in a named encoding such as `base64` or `hex`, and the encoding MUST be declared by the schema
  - replay protection MUST use a unique nonce, issuance timestamp, sequence number, or equivalent binding that is checked against the escrow instance
- `oracle_signature`
  - the schema MUST name the referenced oracle explicitly and MUST bind the decision to that oracle's public key, identifier, or equivalent stable oracle reference
  - anonymous oracle signatures are not sufficient
- `threshold_participant_signatures`
  - the schema MUST define the threshold value and the location of the participant signatures
  - the participant signature set MUST identify each signer and MUST bind each signature to the same escrow-scoped payload
  - the threshold MUST be satisfied by distinct bound participants

`release_rules.release_trigger` states the public condition a specific escrow subtype requires before release; `service.release_decisions` states the generic decision formats the service accepts to satisfy such a trigger. Swap-specific triggers such as `counterparty_fiat_payment_confirmed` remain valid for Pontmore swap flows. For standalone non-swap use, `release_decisions` is the generic vocabulary an application relies on.

### Public/Private Boundary

The `service` block advertises only the public service interface. The following MUST NOT appear in the descriptor:

- wallet identifiers
- custody backend identifiers
- private payment credentials
- internal account details
- private routing state
- operator-internal API keys or bearer secrets

These are operator-layer implementation details. Their absence is what allows the same descriptor to be published openly without exposing operator internals.

### Example: Two-Party Dice Game

A standalone dice game discovers a published escrow descriptor and operates an escrow instance without a Pontmore swap:

```json
{
  "version": 1,
  "escrow_type": "custodial_escrow",
  "networks": ["lightning"],
  "funding_rules": {
    "required_confirmation": "invoice_paid"
  },
  "release_rules": {
    "release_trigger": "application_signed_result",
    "refund_trigger": "timeout_requires_mutual_consent"
  },
  "dispute_rules": {
    "policy": "operator_resolved"
  },
  "reference_format": "bolt11_or_custodial_escrow_reference",
  "custody_authority": "escrow_operator",
  "release_authority": "escrow_operator",
  "refund_authority": "escrow_operator",
  "implementations": [
    {
      "network": "lightning",
      "invoice_asset": "BTC",
      "invoice_currency": "sats",
      "invoice_amount_rule": "exact",
      "payout_network": "lightning"
    }
  ],
  "service": {
    "transport": ["https"],
    "interface": "pontmore_escrow_http_v1",
    "endpoint": "https://escrow.example.com/pontmore/v1",
    "schema_url": "https://escrow.example.com/pontmore/v1/openapi.json",
    "auth": ["nostr_http_auth"],
    "operations": ["create", "funding_instructions", "fund_status", "release", "refund", "cancel"],
    "funding_model": "two_party",
    "release_decisions": ["application_signed_result", "mutual_consent", "operator_decision"]
  },
  "updated_at": 1775559028
}
```

The flow validates the descriptor model:

1. the application discovers the descriptor and reads `service`
2. participant A calls `create` over HTTPS, authenticated with `nostr_http_auth`, and establishes the escrow instance
3. the service issues an invitation or enrollment token for participant B
4. participant B calls `create` with that invitation or enrollment token; the idempotency key, if used, only deduplicates the request
5. the application retrieves `funding_instructions` for the established escrow
6. each participant funds its side; the application reads `fund_status` until both sides are confirmed
7. the application commits a verifiable result (the die roll) and submits an `application_signed_result` release decision
8. if the result is unresolved at timeout, the escrow refunds only after a listed explicit decision such as `mutual_consent` or `operator_decision`; the timeout itself does not authorize refund

## Network Declaration

An escrow descriptor MUST declare every settlement or invoice network supported by the escrow configuration.

The canonical content field is:

- `networks`
  - non-empty array of lowercase network identifiers
  - examples: `bitcoin`, `lightning`

Each value in `networks` SHOULD also be emitted as a repeated `network` tag for relay filtering:

```text
["network", "bitcoin"]
["network", "lightning"]
```

The `networks` content array is the canonical declaration. The repeated tags are an index and discovery aid. Clients MUST NOT treat a repeated `network` tag as supported unless it also appears in `content.networks`.

## Example: Bitcoin and Lightning

```json
{
  "version": 1,
  "escrow_type": "lightning_hold_invoice",
  "networks": ["bitcoin", "lightning"],
  "funding_rules": {
    "required_confirmation": "invoice_held"
  },
  "release_rules": {
    "release_trigger": "counterparty_fiat_payment_confirmed"
  },
  "dispute_rules": {
    "policy": "operator_resolved"
  },
  "reference_format": "bolt11",
  "invoice_network": "lightning",
  "payout_network": "bitcoin",
  "updated_at": 1775559028
}
```

The matching event tags SHOULD include:

```text
["d", "default"]
["network", "bitcoin"]
["network", "lightning"]
```

## Canonical Subtype: `lightning_hold_invoice`

`lightning_hold_invoice` is a canonical escrow subtype for swaps that use a Lightning hold invoice as the escrow lock.

When `escrow_type` is `lightning_hold_invoice`, the descriptor SHOULD include the following additional fields:

- `invoice_network`
- `invoice_asset`
- `invoice_currency`
- `invoice_amount_rule`
- `hold_expiry_rule`
- `settle_authority`
- `cancel_authority`
- `release_rules.release_trigger`
- `release_rules.refund_trigger`
- `preimage_visibility`
- `payout_network`

### Field Intent

- `invoice_network`
  - Lightning network on which the hold invoice is issued
- `invoice_asset`
  - asset locked by the hold invoice, usually bitcoin-denominated Lightning liquidity
- `invoice_currency`
  - invoice denomination convention used by the escrow provider
- `invoice_amount_rule`
  - whether the invoice amount is exact, bounded, or derived from the swap request
- `hold_expiry_rule`
  - timeout rule for unpaid or unresolved hold invoices
- `settle_authority`
  - which participant or operator may settle the invoice
- `cancel_authority`
  - which participant or operator may cancel the invoice
- `release_rules.release_trigger`
  - public condition required before settlement is valid
- `release_rules.refund_trigger`
  - public condition required before cancellation or refund is valid
- `preimage_visibility`
  - whether the preimage is expected to remain operator-local, participant-visible, or public by reference only
- `payout_network`
  - expected payout path after successful release

### Lifecycle Rules

For `lightning_hold_invoice`, the escrow lifecycle SHOULD follow these phases:

1. invoice issued
2. invoice held
3. release condition satisfied
4. settled or canceled

The public swap state machine SHOULD record:

- when the hold invoice becomes the active escrow reference
- when funding is confirmed
- when release authority is exercised
- when cancellation or refund authority is exercised

Raw invoice payloads, settlement secrets, and other sensitive Lightning material SHOULD stay in the companion private message lane unless explicit disclosure is required.

## Canonical Subtype: `custodial_escrow`

`custodial_escrow` is a canonical escrow subtype for swaps where an escrow operator takes custody of the settlement asset or invoice claim and releases or refunds it according to public protocol state.

This subtype is intentionally network-generic. The top-level `networks` array remains the canonical supported-network declaration for the descriptor. Network-specific implementation details MUST be represented under `implementations` and MUST NOT expand the supported network set beyond `content.networks`.

When `escrow_type` is `custodial_escrow`, the descriptor MUST include the following additional fields:

- `custody_authority`
- `release_authority`
- `refund_authority`
- `release_rules.release_trigger`
- `release_rules.refund_trigger`
- `implementations`

### Custodial Field Intent

- `custody_authority`
  - participant or operator role that controls the custodied escrow asset or claim while the swap is active
- `release_authority`
  - participant or operator role that may release the escrow after `release_rules.release_trigger` is satisfied
- `refund_authority`
  - participant or operator role that may refund or cancel the escrow after `release_rules.refund_trigger` is satisfied
- `release_rules.release_trigger`
  - public condition required before release is valid
- `release_rules.refund_trigger`
  - public condition required before refund or cancellation is valid
- `implementations`
  - non-empty array of network-specific implementation profiles

Each implementation entry MUST include:

- `network`
  - one network identifier present in top-level `networks`

Implementation entries MAY include network-specific fields such as:

- `invoice_asset`
- `invoice_currency`
- `invoice_amount_rule`
- `invoice_expiry_rule`
- `payout_network`
- `reference_format`

If an implementation entry includes `reference_format`, it overrides the top-level `reference_format` for that implementation. Otherwise, clients MUST use the top-level `reference_format`.

Clients MUST ignore implementation entries whose `network` value is not present in top-level `networks`. If no valid implementation entries remain, clients MUST treat the descriptor as unusable.

### Example: Lightning Custodial Invoice

```json
{
  "version": 1,
  "escrow_type": "custodial_escrow",
  "networks": ["lightning"],
  "funding_rules": {
    "required_confirmation": "invoice_paid"
  },
  "release_rules": {
    "release_trigger": "counterparty_fiat_payment_confirmed",
    "refund_trigger": "timeout_or_dispute_refund_decision"
  },
  "dispute_rules": {
    "policy": "operator_resolved"
  },
  "reference_format": "bolt11_or_custodial_escrow_reference",
  "custody_authority": "escrow_operator",
  "release_authority": "escrow_operator",
  "refund_authority": "escrow_operator",
  "implementations": [
    {
      "network": "lightning",
      "invoice_asset": "BTC",
      "invoice_currency": "sats",
      "invoice_amount_rule": "derived_from_swap_request",
      "invoice_expiry_rule": "expires_if_unpaid_before_funding_timeout",
      "payout_network": "lightning"
    }
  ],
  "updated_at": 1775559028
}
```

The matching event tags SHOULD include only networks present in `content.networks`:

```text
["d", "default"]
["network", "lightning"]
```

### Custodial Lifecycle Rules

For `custodial_escrow`, the escrow lifecycle SHOULD follow these phases:

1. escrow reference issued
2. custody funded or claim accepted
3. release or refund condition satisfied
4. released, refunded, or canceled

The public swap state machine SHOULD record:

- when the custodial escrow reference becomes active
- when custody funding or claim acceptance is confirmed
- when release authority is exercised
- when refund or cancellation authority is exercised

Raw invoices, private payment instructions, operator account details, and custody internals SHOULD stay in the companion private message lane unless explicit disclosure is required.

## Canonical Subtype: `cashu_escrow`

`cashu_escrow` is a canonical escrow subtype where funds are held as Cashu ecash tokens locked to the escrow operator's pubkey using NUT-11 spending conditions, with a refund pubkey and locktime. It composes on the `custodial_escrow` pattern, with the operator acting as release and refund authority, but the custody is on the bearer instrument itself rather than a relational balance held by the operator.

When `escrow_type` is `cashu_escrow`, the descriptor MUST include the following additional fields:

- `custody_authority`
- `release_authority`
- `refund_authority`
- `release_rules.release_trigger`
- `release_rules.refund_trigger`
- `implementations`

Each implementation entry describing the Cashu escrow lock MUST include:

- `network`
  - one network identifier present in top-level `networks`; for the Cashu escrow lock, MUST be `cashu`
- `mint_url`
  - URL of the Cashu mint used to issue and redeem tokens
- `lock_mechanism`
  - locking primitive; MUST be `p2pk_timelock` (NUT-11 P2PK with locktime and refund pubkey)

Each implementation entry describing the Cashu escrow lock SHOULD include:

- `invoice_expiry_rule`
  - SHOULD be `p2pk_timelock_expiry`
- `reference_format`
  - SHOULD be `cashu_v4_token`
- `payout_network`
  - expected payout path after successful release, typically `lightning`

### Cashu Field Intent

- `network`
  - network identifier for the Cashu implementation entry; MUST be `cashu` and MUST appear in top-level `networks`
- `mint_url`
  - identifies the Cashu mint responsible for issuing and redeeming the locked tokens
- `lock_mechanism`
  - declares the NUT-11 P2PK timelock primitive that binds the tokens to the operator pubkey, refund pubkey, and locktime
- `invoice_expiry_rule`
  - declares that the escrow expires according to the P2PK locktime
- `reference_format`
  - declares that the escrow is referenced as a Cashu v4 token
- `payout_network`
  - declares the expected payout path after successful release

### Example: Cashu P2PK Timelock

```json
{
  "version": 1,
  "escrow_type": "cashu_escrow",
  "networks": ["cashu", "lightning"],
  "funding_rules": {
    "required_confirmation": "cashu_token_locked_to_escrow"
  },
  "release_rules": {
    "release_trigger": "counterparty_fiat_payment_confirmed",
    "refund_trigger": "timeout_or_dispute_refund_decision"
  },
  "dispute_rules": {
    "policy": "operator_resolved"
  },
  "reference_format": "cashu_v4_token",
  "custody_authority": "escrow_operator",
  "release_authority": "escrow_operator",
  "refund_authority": "escrow_operator",
  "implementations": [
    {
      "network": "cashu",
      "invoice_asset": "BTC",
      "invoice_currency": "sats",
      "invoice_amount_rule": "derived_from_swap_request",
      "invoice_expiry_rule": "p2pk_timelock_expiry",
      "payout_network": "lightning",
      "mint_url": "https://mint.example.com",
      "lock_mechanism": "p2pk_timelock",
      "reference_format": "cashu_v4_token"
    }
  ],
  "updated_at": 1775559028
}
```

The matching event tags SHOULD include only networks present in `content.networks`:

```text
["d", "default"]
["network", "cashu"]
["network", "lightning"]
```

### Cashu Lifecycle Rules

For `cashu_escrow`, the escrow lifecycle SHOULD follow these phases:

1. locked token reference issued
2. lock verified against declared mint and operator pubkey
3. release or refund condition satisfied
4. released, refunded, or locktime-expired

The public swap state machine SHOULD record:

- when the locked token reference becomes active
- when funding lock is confirmed
- when release authority is exercised
- when refund or cancellation authority is exercised

Raw Cashu token strings, mint credentials, and preimages SHOULD stay in the companion private message lane. Only opaque references or hashes SHOULD appear in public evidence events.

## Selection Rules

Every agent profile should declare:

- at least one usable escrow configuration
- one default escrow configuration

That declared escrow must be usable without out-of-band negotiation at swap time.

For standalone (non-swap) use, an application SHOULD select a descriptor whose `service` block is present and whose advertised `transport`, `endpoint`, `auth`, `interface`, `operations`, `funding_model`, `release_decisions`, and `schema_url` all match the application's supported capabilities and trust constraints. A descriptor without `service` MUST NOT be treated as standalone-sufficient.

## Open Questions

Additional escrow mechanisms beyond `lightning_hold_invoice`, `custodial_escrow`, and `cashu_escrow` may still need their own canonical subtype-specific schemas.

`cashu_escrow` introduces a dependency on a specific Cashu mint. Mint trust, mint selection, and mint failure modes are implementation assumptions rather than Pontmore protocol state. [Nimdolf](https://github.com/cashubtc/nuts/pull/390) is a possible future direction for mint liveness failover, but this proposal does not bind to it.

Open questions for the service interface:

- additional escrow mechanisms beyond `lightning_hold_invoice`, `custodial_escrow`, and `cashu_escrow` may still need their own canonical subtype-specific schemas
