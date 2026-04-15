# Koha design patterns

Documented patterns used across the Koha codebase. These are not abstract guidelines — they describe concrete classes and conventions that exist in the code today.

## Availability + Policy pattern

### Overview

The Availability + Policy pattern separates circulation validation into three layers:

```
Policy classes          → "What is the effective rule?"
Availability classes    → "Given the rules, what is blocked/warned/confirmed?"
Action functions        → "Do the thing (AddReturn, AddIssue, etc.)"
```

Each layer has a single responsibility. Policy classes resolve effective configuration values. Availability classes orchestrate multiple policy checks and produce a standardized result. Action functions consume the result and execute the operation.

### Koha::Availability::Result

The result envelope used by all availability checks. Categorizes conditions into:

- **blockers** — prevent the action (e.g., item is withdrawn and `BlockReturnOfWithdrawnItems` is on)
- **confirmations** — require user acknowledgment (e.g., item is not checked out)
- **warnings** — informational, don't prevent the action (e.g., item is withdrawn but check-in is allowed)
- **context** — related objects for the caller (e.g., checkout, patron)

```perl
my $result = Koha::Availability::Result->new();

$result->add_blocker( BlockedWithdrawn => 1 );
$result->add_confirmation( NotIssued => $barcode );
$result->add_warning( withdrawn => 1 );
$result->set_context( checkout => $checkout );

if ( $result->available ) {
    # no blockers — proceed
}
```

### Availability classes

Operation-specific orchestrators that run checks and populate a Result object. Named `Koha::<Object>::<Operation>::Availability`.

**Existing:**

- `Koha::Item::Checkin::Availability` — check-in validation (bug 41728)

**Planned:**

- `Koha::Patron::Checkout::Availability` — patron-side checkout eligibility (bug 42389)
- `Koha::Patron::Hold::Availability` — patron-side hold eligibility
- `Koha::Item::Hold::Availability` — item-level hold policy (bug 42386)

#### no_short_circuit parameter

Availability classes support a `no_short_circuit` parameter following the pattern established by `Koha::Patron->can_place_holds`:

```perl
# Default: short-circuit on first blocker (efficient for AddReturn)
my $availability = $item->checkin_availability(
    { library => $branch }
);

# Collect all blockers (for API responses)
my $availability = $item->checkin_availability(
    { library => $branch, no_short_circuit => 1 }
);
```

The default short-circuits because callers like `AddReturn` only act on the first blocker — running all checks wastes DB queries. API consumers pass `no_short_circuit => 1` to get the full picture in a single response.

### Policy classes

Stateless classes that resolve effective configuration values from system preferences, circulation rules, and patron/item attributes. Named `Koha::Policy::<Domain>`.

**Existing:**

- `Koha::Policy::Holds` — resolves which library controls hold rules (`holds_control_library`)
- `Koha::Policy::Patrons::Cardnumber` — validates cardnumber format and uniqueness

**Planned:**

- `Koha::Policy::Circulation` — resolve circ control library, replacing `_GetCircControlBranch` (bug 42385)
- `Koha::Policy::Patrons::ChargeLimits` — resolve effective charge limits from category/syspref (bug 42388)
- `Koha::Policy::Returns` — resolve return branch policy (`AllowReturnToBranch`) and transfer limits, currently `Koha::Item->can_be_returned_at`

#### When to create a Policy class

Create a Policy class when the resolution logic involves multiple inputs (item + patron + syspref + circ rules). Simple boolean syspref checks (`BlockReturnOfWithdrawnItems`, `BlockReturnOfLostItems`) don't need a Policy class — the Availability orchestrator reads them directly.

### How the layers connect

Example: check-in flow

```
Koha::Item::Checkin::Availability::check()
    │
    ├── reads BlockReturnOfWithdrawnItems syspref directly
    ├── reads BlockReturnOfLostItems syspref directly
    ├── calls $item->can_be_returned_at()  ← policy resolution
    │       └── reads AllowReturnToBranch syspref
    │       └── checks branch transfer limits
    │       └── candidate for Koha::Policy::Returns
    │
    └── returns Koha::Availability::Result
            │
            └── consumed by AddReturn in C4::Circulation
```

### Relationship to existing code

The pattern is an evolution of existing approaches:

| Old approach | New approach |
|---|---|
| `Koha::Patron->can_place_holds` (returns `Koha::Result::Boolean`) | Would become `Koha::Patron::Hold::Availability` (returns `Koha::Availability::Result`) |
| `Koha::Patron->can_checkout` (returns raw hashref) | Would become `Koha::Patron::Checkout::Availability` (returns `Koha::Availability::Result`) |
| `C4::Reserves::CanItemBeReserved` (returns hashref with `{status}`) | Would become `Koha::Item::Hold::Availability` |
| `C4::Circulation::CanBookBeIssued` (returns multiple hashrefs) | Future `Koha::Item::Checkout::Availability` |
| `_GetCircControlBranch` (private exported function) | `Koha::Policy::Circulation->circ_control_library` |


## Koha::Result::Boolean

A boolean result object that carries structured error messages. Use it for methods that answer a yes/no question but need to explain *why not*.

### When to use

Use `Koha::Result::Boolean` for validation methods that:
- Return true/false
- Need to report one or more reasons for failure
- Are consumed by callers that branch on the boolean but may also inspect the reason

Typical use cases: "is this valid?", "can this patron do X?", "is this item safe to delete?"

### When NOT to use

Don't use it when:
- A plain boolean suffices (no caller needs the reason)
- The result has multiple categories (blockers/warnings/confirmations) — use `Koha::Availability::Result` instead
- The method performs an action rather than answering a question

### API

```perl
use Koha::Result::Boolean;

# Success
return Koha::Result::Boolean->new(1);

# Failure with reason
return Koha::Result::Boolean->new(0)->add_message(
    { message => 'already_exists' }
);

# Failure with payload
return Koha::Result::Boolean->new(0)->add_message(
    {
        message => 'debt_limit',
        type    => 'error',
        payload => { total => $outstanding, max => $max }
    }
);
```

The object overloads `bool`, so callers can use it directly in conditionals:

```perl
my $result = $patron->can_place_holds;

if ( $result ) {
    # patron can place holds
} else {
    my $reason = $result->messages->[0]->message;    # e.g. 'expired'
    my $payload = $result->messages->[0]->payload;   # e.g. { total => 50, max => 25 }
}
```

### Existing usage

- `Koha::Patron->can_place_holds` — patron hold eligibility with `no_short_circuit` and `overrides` support
- `Koha::Policy::Patrons::Cardnumber->is_valid` — cardnumber validation (`already_exists`, `invalid_length`)
- `Koha::Item->safe_to_delete` — item deletion safety check
- `Koha::CurbsidePickupPolicy->is_valid_pickup_datetime` — pickup slot validation

### Relationship to Koha::Availability::Result

`Koha::Result::Boolean` and `Koha::Availability::Result` serve different purposes:

| | `Result::Boolean` | `Availability::Result` |
|---|---|---|
| Answer | yes/no | available, with details |
| Categories | flat message list | blockers, confirmations, warnings, context |
| Overloads bool | yes | via `available()` method |
| Use for | validation gates | operation pre-checks |

`can_place_holds` uses `Result::Boolean` because it's a patron-level gate: "can this patron place holds at all?" The future `Koha::Item::Hold::Availability` would use `Availability::Result` because it orchestrates multiple checks with different severity levels for a specific hold operation.

### Tracking bugs

- Bug 41728: Checkin availability (implemented)
- Bug 42385: Extract `_GetCircControlBranch` into `Koha::Policy::Circulation`
- Bug 42386: Unify hold availability checks
- Bug 42387: [UMBRELLA] Extend Availability pattern to checkout and holds
- Bug 42388: Extract patron charge limits into `Koha::Policy::Patrons::ChargeLimits`
- Bug 42389: Refactor `Koha::Patron->can_checkout` into Availability pattern
