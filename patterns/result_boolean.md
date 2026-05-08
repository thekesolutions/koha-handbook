# Result::Boolean

## Overview

A boolean result object that carries structured error messages. Use it for methods that answer a yes/no question but need to explain *why not*.

## When to use

Use `Koha::Result::Boolean` for validation methods that:
- Return true/false
- Need to report one or more reasons for failure
- Are consumed by callers that branch on the boolean but may also inspect the reason

Typical use cases: "is this valid?", "can this patron do X?", "is this item safe to delete?"

## When NOT to use

Don't use it when:
- A plain boolean suffices (no caller needs the reason)
- The result has multiple categories (blockers/warnings/confirmations) — use `Koha::Result::Availability` instead
- The method performs an action rather than answering a question

## API

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

## Existing usage

- `Koha::Patron->can_place_holds` — patron hold eligibility with `no_short_circuit` and `overrides` support
- `Koha::Policy::Patrons::Cardnumber->is_valid` — cardnumber validation (`already_exists`, `invalid_length`)
- `Koha::Item->safe_to_delete` — item deletion safety check
- `Koha::CurbsidePickupPolicy->is_valid_pickup_datetime` — pickup slot validation

## Relationship to Koha::Result::Availability

| | `Result::Boolean` | `Result::Availability` |
|---|---|---|
| Answer | yes/no | available, with details |
| Categories | flat message list | blockers, confirmations, warnings, context |
| Overloads bool | yes | via `available()` method |
| Use for | validation gates | operation pre-checks |

`can_place_holds` uses `Result::Boolean` because it's a patron-level gate: "can this patron place holds at all?" `Koha::Item::Availability::Hold` uses `Result::Availability` because it orchestrates multiple checks with different severity levels for a specific hold operation.
