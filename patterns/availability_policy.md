# Availability + Policy

## Overview

The Availability + Policy pattern separates circulation validation into three layers:

```
Policy classes          → "What is the effective rule?"
Availability classes    → "Given the rules, what is blocked/warned/confirmed?"
Action functions        → "Do the thing (AddReturn, AddIssue, etc.)"
```

Each layer has a single responsibility. Policy classes resolve effective configuration values. Availability classes orchestrate multiple policy checks and produce a standardized result. Action functions consume the result and execute the operation.

## Koha::Result::Availability

The result envelope used by all availability checks. Categorizes conditions into:

- **blockers** — prevent the action (e.g., item is withdrawn and `BlockReturnOfWithdrawnItems` is on)
- **confirmations** — require user acknowledgment (e.g., item is not checked out)
- **warnings** — informational, don't prevent the action (e.g., item is withdrawn but check-in is allowed)
- **context** — related objects for the caller (e.g., checkout, patron)

```perl
my $result = Koha::Result::Availability->new();

$result->add_blocker( BlockedWithdrawn => 1 );
$result->add_confirmation( NotIssued => $barcode );
$result->add_warning( withdrawn => 1 );
$result->set_context( checkout => $checkout );

if ( $result->available ) {
    # no blockers — proceed
}
```

## Availability classes

Operation-specific orchestrators that run checks and populate a Result object. Named `Koha::<Object>::Availability::<Operation>`.

**Existing:**

- `Koha::Item::Availability::Checkin` — check-in validation (Bug 41728)
- `Koha::Patron::Availability::Hold` — patron-level hold eligibility (Bug 42386)
- `Koha::Item::Availability::Hold` — item-level hold policy (Bug 42386)
- `Koha::Biblio::Availability::Hold` — biblio-level hold availability (Bug 42386)

**Planned:**

- `Koha::Patron::Availability::Checkout` — patron-side checkout eligibility (Bug 42389)

### no_short_circuit parameter

Availability classes support a `no_short_circuit` parameter:

```perl
# Default: short-circuit on first blocker (efficient for AddReturn)
my $availability = Koha::Item::Availability::Hold->check(
    { item => $item, patron => $patron }
);

# Collect all blockers (for API responses)
my $availability = Koha::Item::Availability::Hold->check(
    { item => $item, patron => $patron, no_short_circuit => 1 }
);
```

The default short-circuits because callers like `AddReturn` only act on the first blocker — running all checks wastes DB queries. API consumers pass `no_short_circuit => 1` to get the full picture in a single response.

## Policy classes

Stateless classes that resolve effective configuration values from system preferences, circulation rules, and patron/item attributes. Named `Koha::Policy::<Domain>`.

**Existing:**

- `Koha::Policy::Holds` — resolves which library controls hold rules (`holds_control_library`)
- `Koha::Policy::Patrons::Cardnumber` — validates cardnumber format and uniqueness

**Planned:**

- `Koha::Policy::Circulation` — resolve circ control library, replacing `_GetCircControlBranch` (Bug 42385)
- `Koha::Policy::Patrons::ChargeLimits` — resolve effective charge limits from category/syspref (Bug 42388)
- `Koha::Policy::Returns` — resolve return branch policy (`AllowReturnToBranch`) and transfer limits

### When to create a Policy class

Create a Policy class when the resolution logic involves multiple inputs (item + patron + syspref + circ rules). Simple boolean syspref checks (`BlockReturnOfWithdrawnItems`, `BlockReturnOfLostItems`) don't need a Policy class — the Availability orchestrator reads them directly.

## How the layers connect

Example: check-in flow

```
Koha::Item::Availability::Checkin::check()
    │
    ├── reads BlockReturnOfWithdrawnItems syspref directly
    ├── reads BlockReturnOfLostItems syspref directly
    ├── calls $item->can_be_returned_at()  ← policy resolution
    │       └── reads AllowReturnToBranch syspref
    │       └── checks branch transfer limits
    │       └── candidate for Koha::Policy::Returns
    │
    └── returns Koha::Result::Availability
            │
            └── consumed by AddReturn in C4::Circulation
```

Example: hold flow

```
Koha::Item::Availability::Hold::check()
    │
    ├── column checks (damaged, item fields)
    ├── DB queries (item_already_on_hold, recall)
    ├── Koha::Policy::Holds->holds_control_library()
    ├── Koha::CirculationRules (holdallowed, reservesallowed)
    ├── chains Koha::Patron::Availability::Hold (counts)
    ├── pickup validation (last — most expensive)
    │
    └── returns Koha::Result::Availability
            │
            └── consumed by CanItemBeReserved wrapper
```

## Relationship to existing code

| Old approach | New approach |
|---|---|
| `C4::Reserves::CanItemBeReserved` (returns hashref with `{status}`) | `Koha::Item::Availability::Hold` |
| `C4::Reserves::CanBookBeReserved` (returns hashref with `{status}`) | `Koha::Biblio::Availability::Hold` |
| `Koha::Patron->can_place_holds` (returns `Koha::Result::Boolean`) | `Koha::Patron::Availability::Hold` |
| `Koha::Patron->can_checkout` (returns raw hashref) | Future `Koha::Patron::Availability::Checkout` |
| `C4::Circulation::CanBookBeIssued` (returns multiple hashrefs) | Future `Koha::Item::Availability::Checkout` |
| `_GetCircControlBranch` (private exported function) | `Koha::Policy::Circulation->circ_control_library` |

## Tracking bugs

- Bug 41728: Checkin availability (implemented)
- Bug 42385: Extract `_GetCircControlBranch` into `Koha::Policy::Circulation`
- Bug 42386: Unify hold availability checks (implemented)
- Bug 42387: [UMBRELLA] Extend Availability pattern to checkout and holds
- Bug 42388: Extract patron charge limits into `Koha::Policy::Patrons::ChargeLimits`
- Bug 42389: Refactor `Koha::Patron->can_checkout` into Availability pattern
