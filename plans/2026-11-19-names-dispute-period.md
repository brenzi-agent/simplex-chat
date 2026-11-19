# Names Conditions: dispute period

Repository: simplex-chat/simplex-chat.
Branch: `ab/names-dispute-period`.
Base: master `44a193b54` (#8266).

The user's words:

> registration can define a "dispute period" during which the guardian role can unregister a name

> renewals are final

> also submit a PR to change the name terms accordingly

## Problem

A holder accepts docs/NAMES-CONDITIONS.md on chain. It states that changing a name's record and subnames requires the holder's key, and that a name can be extended until 90 days after its period ends.

The registry contracts change on simplex-network/ens-contracts, branch `ab/dispute-period`:

- a name that SimpleX Chat Ltd registers for an order has a dispute period of 14 to 120 days, stored in the registry;
- before it ends, the registry's guardian or admin can end the registration, with its record and subnames;
- before it ends, the name cannot be extended;
- a name registered by the holder's own Ethereum transaction has none.

The conditions would describe a registry that holders do not get.

## Registration states

```mermaid
stateDiagram-v2
  [*] --> disputable: registered by SimpleX Chat Ltd, other than a reserved name
  [*] --> final: registered by the holder's own Ethereum transaction, or a reserved name assigned by the registry's admin
  disputable --> disputable: transferred
  disputable --> final: dispute period ends
  disputable --> available: ended, without a valid order, or with a payment disputed, reversed or refunded in full
  final --> final: extended, transferred
  final --> no_extension: a payment for the name disputed, reversed or refunded in full
  final --> available: 90 days after the registration period
  no_extension --> reserved: 90 days after the registration period
  reserved --> [*]: assigned by the registry's admin
  available --> [*]
```

## Changes

docs/NAMES-CONDITIONS.md, section "What the registry allows":

| Bullet | Change |
|---|---|
| your key | unchanged |
| dispute period, new, second | which registrations have one, its length, what the registry allows before its end, the only grounds for ending, availability after, transfers |
| period and extension | extension after the dispute period, if any; "the period" named "the registration period" |
| refused extensions, new | after a payment for the name is disputed, reversed or refunded in full and its registration is not ended |

The other sections are unchanged. A separate change, for the registry's upgradeable contracts, edits the section "Registry"; the change merged second is rebased.

## After the merge

The name policy's general and community terms links are set to this file at the merge commit, in the same governance batch as `setSponsor` of the sponsor that takes `disputeDays`, so that every registration with a dispute period is made under this version.
