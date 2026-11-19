# Names Conditions: dispute period

Repository: simplex-chat/simplex-chat.
Branch: `ab/names-dispute-period`.
Base: master `44a193b54` (#8266).

Requirement: the Names Conditions describe the registry's dispute period, as the registry contracts implement it.

## Problem

A holder accepts docs/NAMES-CONDITIONS.md on chain. It states that changing the record of a name, creating subnames and transferring control require the holder's key, and that a name can be extended until 90 days after its period ends.

The registry contracts change on simplex-network/ens-contracts, branch `ab/dispute-period`:

- a registration through a registrar funded on the sponsor has a dispute period of 14 to 120 days, its end stored in the registry;
- before that end, the registry's guardian or admin can end the registration, with its record;
- before that end, the name cannot be extended;
- a registration paid in ETH by the registrant's own transaction, or assigned by the registry's admin, is not subject to a dispute period.

The conditions would describe a registry that holders do not get.

## Registration states

| State | Entered by | What the registry allows |
|---|---|---|
| disputable | registration through a funded registrar | transfer; end by the guardian or admin; reservation |
| final | registration paid in ETH by the registrant; assignment by the admin; end of the dispute period | transfer; extension; reservation |
| no_extension | reservation | transfer until the registration period ends; end by the guardian or admin while still disputable |
| withheld | 90 days after the registration period of a reserved name | assignment by the admin |
| available | end of a registration; 90 days after the registration period of an unreserved name | registration |

```mermaid
stateDiagram-v2
  [*] --> disputable: registered through a funded registrar
  [*] --> final: paid in ETH by the registrant, or assigned by the admin
  disputable --> disputable: transferred
  disputable --> final: dispute period ends
  disputable --> available: ended on a ground the conditions state
  disputable --> no_extension: reserved
  final --> final: extended or transferred
  final --> no_extension: reserved
  final --> available: 90 days after the registration period
  no_extension --> available: ended, while disputable
  no_extension --> withheld: 90 days after the registration period
  withheld --> final: assigned by the admin
  available --> [*]
```

## Changes

docs/NAMES-CONDITIONS.md, section "What the registry allows":

| Bullet | Change |
|---|---|
| your key | the next point as its exception |
| dispute period | new, second |
| period and extension | after the dispute period, if any; "the period" named "the registration period" |
| refused extensions | new, last |

The dispute period bullet states:

- which registrations have one, its length, and its end in the registry;
- what the registry allows before its end;
- the only grounds for ending a registration;
- the name's availability after;
- that a transfer leaves it unchanged.

The other sections are unchanged. A separate change, for the registry's upgradeable contracts, edits the section "Registry"; the change merged second merges master.

## After the merge

The website publishes docs/*.md on each push to master (web.yml), so the text appears there at the merge. Holders accept the version linked on chain: the name policy's general and community terms links are set to this file at the merge commit of the later of this change and the change for the upgradeable contracts:

1. tagged `general-terms-vN` and `community-terms-vN`;
2. mirrored on git.simplex.chat;
3. set in the same governance batch as `setSponsor` of the sponsor that takes `disputeDays`, so that every registration with a dispute period is made under this version.
