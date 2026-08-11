# PIP-01: Escrow Descriptor

## Status

- Status: Draft
- Implementation: Required
- Scope: public escrow interop descriptor for discovery, compatibility, and service schema discovery
- Related:
  - [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
  - [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md)
  - [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md)

## Purpose

This document defines the public escrow descriptor event referenced by agent definitions and swaps.

An escrow descriptor is a compatibility object. It declares the public facts needed to discover an escrow configuration, decide whether a client can use it, reference it from Pontmore swap flows, and locate the service schema that defines the callable escrow interface.

PIP-01 does not define an escrow service's internal state machine, endpoint contract, authentication protocol, request payloads, response payloads, error format, or operation-specific authorization rules. When a descriptor advertises a callable service, those details belong to the referenced service schema.

## Event Type

- kind: `30361`
- addressable
- `d` tag: stable identifier for one escrow configuration

## Function

The escrow descriptor tells counterparties and operators:

- what escrow mechanism is used
- which settlement or invoice networks are supported
- what public funding and release rules are advertised
- how the escrow instance is referenced
- what timeout and dispute assumptions apply
- where to find the service schema, if the escrow can be invoked directly

The public swap lifecycle remains defined by [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md). Dispute and timeout fallback policy remains defined by [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md).

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

## Descriptor Use Levels

An escrow descriptor has two intended use levels:

- **compatibility and discovery** - the descriptor declares which networks, assets, reference formats, funding rules, release rules, and dispute policy an escrow supports, so that agents and swaps can select it
- **service schema discovery** - the descriptor additionally points to a machine-readable schema that defines the callable escrow contract

A descriptor that omits the `service` block is sufficient for compatibility and discovery and for use inside Pontmore swap flows where the swap state machine in [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md) carries public execution state.

A descriptor that includes a `service` block does not make PIP-01 the source of truth for invocation behavior. The descriptor only advertises where the invocation schema is found and what schema language it uses. Clients MUST validate the referenced schema before treating the escrow as directly callable.

## Service Schema

The optional `service` content field points to the machine-readable schema that defines the callable escrow interface.

When `service` is present, it MUST include:

- `schema`
  - object containing the service schema pointer
- `schema.type`
  - schema language identifier
  - initial supported values are `openapi` and `asyncapi`
- `schema.url`
  - absolute `https://` URL for the schema artifact

Initial PIP-01 schema types are intentionally limited to:

- `openapi`
- `asyncapi`

`smithy`, `protobuf`, and generic `json` schema pointers are out of scope for the initial registry. They MAY be added by a later PIP or revision if there is concrete interoperability need.

### Schema Requirements

The referenced schema MUST define the full callable interface, including:

- transport or protocol
- endpoint or server information
- authentication
- operations
- request payloads
- response payloads
- error format
- operation-specific authorization rules

If the service exposes status values, operation names, funding flows, release decisions, split outcomes, idempotency keys, or participant-binding rules, those details MUST be defined by the referenced schema. They are not canonical PIP-01 protocol state.

### Schema Fetch Safety

Clients MUST apply schema fetch safety checks before dereferencing `service.schema.url`.

The schema URL:

- MUST use `https://`
- MUST NOT resolve to private, loopback, link-local, multicast, or otherwise unsafe network destinations
- SHOULD resolve to an immutable or versioned schema artifact

Clients SHOULD use bounded fetches, redirect limits, content-type checks, and response-size limits when retrieving schemas.

Clients MAY reject unsupported schema types, unsupported schema-language versions, unsafe schema URLs, mutable schema URLs, or schemas whose callable interface does not match the client's trust and capability requirements.

## Public/Private Boundary

The descriptor is public protocol state. It MUST NOT include:

- wallet identifiers
- custody backend identifiers
- private payment credentials
- internal account details
- private routing state
- operator-internal API keys or bearer secrets
- private payment instructions
- raw invoices
- raw Cashu token strings
- settlement secrets
- internal review notes

These are operator-layer implementation details or private execution payloads. Their absence is what allows the same descriptor to be published openly without exposing operator internals.

Pontmore implementations SHOULD carry non-public execution payloads through private channels such as the companion Gift Wrap lane described in [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md).

## Network Declaration

An escrow descriptor MUST declare every settlement or invoice network supported by the escrow configuration.

The canonical content field is:

- `networks`
  - non-empty array of lowercase network identifiers
  - examples: `bitcoin`, `lightning`, `cashu`

Each value in `networks` SHOULD also be emitted as a repeated `network` tag for relay filtering:

```text
["network", "bitcoin"]
["network", "lightning"]
```

The `networks` content array is the canonical declaration. The repeated tags are an index and discovery aid. Clients MUST NOT treat a repeated `network` tag as supported unless it also appears in `content.networks`.

## Funding Rules

`funding_rules` declares descriptor-level compatibility facts about how an escrow expects funding to be satisfied.

Escrow funding is described as an `m of n` requirement.

- `funding_rules.funding_threshold` is `m`: the minimum number of declared funding participants whose funding must be confirmed before the escrow is considered funded
- `funding_rules.participant_count` is `n`: the total number of declared funding participants for the escrow

`funding_threshold` MUST be an integer greater than or equal to `1`.

`participant_count` MUST be an integer greater than or equal to `funding_threshold`.

A `1 of 1` funding rule represents a single-funder escrow. A `2 of 2` funding rule represents a two-party escrow where both declared funders must fund. A `1 of 2` funding rule represents one participant funding on behalf of a two-party escrow. Other values represent threshold funding.

The descriptor declares only the funding cardinality required for compatibility. PIP-01 does not define a canonical standalone funding state machine. The referenced service schema MUST define how participants are identified, how funding instructions are retrieved, how partial funding is represented, and how timeout cancellation or refund works.

If `participant_count` is greater than `1`, the descriptor or referenced service schema MUST define how partially funded escrows can be canceled or refunded after timeout. A descriptor MUST NOT imply that partially funded capital can remain locked indefinitely with no timeout or fallback path.

## Release Rules

`release_rules` declares descriptor-level compatibility facts about the public condition required before release or refund is valid.

Release modes MAY be advertised as compatibility hints. When present, they SHOULD use a small vocabulary such as:

- `mutual_consent`
- `operator_decision`
- `external_attestation`

Exact payloads, signer identities, signature encodings, thresholds, replay protection, result binding, and operation-specific authorization rules belong to the referenced service schema.

PIP-01 does not define `split` as a canonical operation. If a service supports partial outcomes, its OpenAPI or AsyncAPI schema can expose and define that operation. PIP-03 may still describe dispute resolution as splitting an outcome where escrow policy allows it.

## Dispute Rules and Timeout Fallback

`dispute_rules` declares descriptor-level compatibility facts about the applicable dispute policy.

Timeout and refund fallback metadata advertised by an escrow descriptor MUST be compatible with [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md). PIP-03 is the source of truth for timeout classes and fallback resolution policy.

When a descriptor advertises a timeout class, the descriptor or referenced service schema MUST identify the applicable fallback resolution required by PIP-03. A `mutual_consent`-only timeout path with no fallback is not a valid terminal policy.

## Reference Format

`reference_format` declares how swaps and clients refer to an escrow instance or escrow claim.

Examples include:

- `bolt11`
- `bolt11_or_custodial_escrow_reference`
- `cashu_v4_token`
- opaque service-defined references

If network-specific implementation entries override the top-level `reference_format`, clients MUST use the implementation-specific value only for that implementation.

## Descriptor Example

```json
{
  "version": 1,
  "escrow_type": "custodial_escrow",
  "networks": ["bitcoin", "lightning"],
  "funding_rules": {
    "funding_threshold": 1,
    "participant_count": 1,
    "required_confirmation": "invoice_paid",
    "funding_timeout": "funding timeout"
  },
  "release_rules": {
    "release_trigger": "counterparty_fiat_payment_confirmed",
    "refund_trigger": "timeout_or_dispute_refund_decision",
    "release_modes": ["mutual_consent", "operator_decision"]
  },
  "dispute_rules": {
    "policy": "operator_resolved",
    "timeout_fallback": "operator_decision"
  },
  "reference_format": "bolt11_or_custodial_escrow_reference",
  "service": {
    "schema": {
      "type": "openapi",
      "url": "https://escrow.example.com/pontmore-escrow.openapi.json"
    }
  },
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

When `escrow_type` is `lightning_hold_invoice`, the descriptor SHOULD include descriptor-level compatibility facts for:

- invoice network
- invoice asset or currency convention
- invoice amount rule
- hold expiry rule
- settlement authority
- cancellation authority
- release trigger
- refund trigger
- preimage visibility boundary
- payout network

Raw invoice payloads, settlement secrets, preimages, and private payout instructions SHOULD stay out of the public descriptor.

## Canonical Subtype: `custodial_escrow`

`custodial_escrow` is a canonical escrow subtype for swaps where an escrow operator takes custody of the settlement asset or invoice claim and releases or refunds it according to public protocol state or the referenced service schema.

This subtype is network-generic. The top-level `networks` array remains the canonical supported-network declaration for the descriptor. Network-specific implementation details MUST be represented under `implementations` and MUST NOT expand the supported network set beyond `content.networks`.

When `escrow_type` is `custodial_escrow`, the descriptor SHOULD include descriptor-level compatibility facts for:

- custody authority
- release authority
- refund authority
- release trigger
- refund trigger
- network-specific implementation profiles

Each implementation entry MUST identify one `network` value that is present in top-level `networks`. Clients MUST ignore implementation entries whose `network` value is not present in top-level `networks`. If no valid implementation entries remain, clients MUST treat the descriptor as unusable.

Raw invoices, private payment instructions, operator account details, custody internals, and private reconciliation records SHOULD stay out of the public descriptor.

## Canonical Subtype: `cashu_escrow`

`cashu_escrow` is a canonical escrow subtype where funds are held as Cashu ecash tokens locked to the escrow operator's pubkey using NUT-11 spending conditions, with a refund pubkey and locktime.

When `escrow_type` is `cashu_escrow`, the descriptor SHOULD include descriptor-level compatibility facts for:

- custody authority
- release authority
- refund authority
- release trigger
- refund trigger
- Cashu network implementation profile
- mint URL
- lock mechanism
- token reference format
- payout network

Each implementation entry describing the Cashu escrow lock MUST identify a `network` value of `cashu`, and `cashu` MUST appear in top-level `networks`.

Raw Cashu token strings, mint credentials, preimages, and private payout instructions SHOULD stay out of the public descriptor. Only opaque references or hashes SHOULD appear in public evidence events unless targeted disclosure is required by the applicable dispute policy.

## Selection Rules

Every agent profile should declare:

- at least one usable escrow configuration
- one default escrow configuration

That declared escrow must be usable without out-of-band negotiation at swap time.

For direct service invocation, an application SHOULD select a descriptor whose `service.schema.type` is supported, whose `service.schema.url` passes fetch-safety checks, and whose referenced schema matches the application's supported capabilities and trust constraints.

A descriptor without `service.schema` MUST NOT be treated as directly callable solely from PIP-01.

## Open Questions

1. **Additional schema types**
   - Should future revisions add `smithy`, `protobuf`, or another schema type once a concrete implementation needs it?

2. **Descriptor-level release-mode hints**
   - Is the small compatibility vocabulary of `mutual_consent`, `operator_decision`, and `external_attestation` sufficient, or should PIP-01 avoid descriptor-level release-mode hints entirely?

3. **Funding cardinality metadata**
   - What additional descriptor-level metadata, if any, is needed for clients to reject unsafe multi-party funding before fetching the service schema?

4. **Custodial accountability references**
   - Should `custodial_escrow` descriptors advertise a public accountability reference, such as proof of reserve, attestation, or collateral, without leaking operator-private implementation details?
