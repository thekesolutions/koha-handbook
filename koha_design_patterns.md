# Koha design patterns

Documented patterns used across the Koha codebase. These are not abstract guidelines — they describe concrete classes and conventions that exist in the code today.

## Pattern catalog

| Pattern | Description | Key classes |
|---------|-------------|-------------|
| [Availability + Policy](patterns/availability_policy.md) | Separates circulation validation into Policy (resolve rules), Availability (check blockers), and Action (execute) layers | `Koha::Result::Availability`, `Koha::Item::Availability::Hold`, `Koha::Policy::Holds` |
| [Result::Boolean](patterns/result_boolean.md) | Boolean result object that carries structured "why not" messages | `Koha::Result::Boolean` |
| [Metadata Extractor](patterns/metadata_extractor.md) | MARC-flavor-aware extraction of structured data from bibliographic records | `Koha::Biblio::Metadata::Extractor`, `::MARC::MARC21`, `::MARC::UNIMARC` |
| [Record Collections](patterns/record_collections.md) | Mixin for serializing Koha::Objects result sets into MARC formats (MARCXML, MiJ, USMARC) | `Koha::Objects::Record::Collections` |

## Choosing the right result type

```
Is the answer yes/no with optional reasons?
  → Koha::Result::Boolean

Does the check produce blockers, confirmations, warnings, and context?
  → Koha::Result::Availability

Is it a simple boolean with no explanation needed?
  → plain 1/0
```

## Choosing where to put MARC logic

```
Does it extract data from MARC fields that differ by flavor?
  → Koha::Biblio::Metadata::Extractor::MARC::<Flavor>

Is it a convenience accessor on the Koha::Biblio object?
  → Method on Koha::Biblio that uses the Extractor internally

Is it MARC manipulation (adding/removing fields)?
  → RecordProcessor filters or direct MARC::Record usage
```
