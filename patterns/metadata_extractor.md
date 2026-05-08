# Metadata Extractor

## Overview

The `Koha::Biblio::Metadata::Extractor` pattern provides a MARC-flavor-aware way to extract structured data from bibliographic records. It dispatches to flavor-specific subclasses (MARC21, UNIMARC) so callers never need to know which MARC tags to read.

## Class hierarchy

```
Koha::Biblio::Metadata::Extractor          (factory — returns flavor-specific instance)
  └── Koha::Biblio::Metadata::Extractor::MARC        (base, shared logic)
        ├── Koha::Biblio::Metadata::Extractor::MARC::MARC21
        └── Koha::Biblio::Metadata::Extractor::MARC::UNIMARC
```

## Usage

```perl
use Koha::Biblio::Metadata::Extractor;

# From a Koha::Biblio object
my $extractor = Koha::Biblio::Metadata::Extractor->new({ biblio => $biblio });

# Or from a raw MARC::Record
my $extractor = Koha::Biblio::Metadata::Extractor->new({ metadata => $marc_record });

# Call flavor-agnostic methods
my $upc  = $extractor->get_normalized_upc;
my $oclc = $extractor->get_normalized_oclc;
my $ctrl = $extractor->get_control_number;
```

## How it works

1. The `Extractor->new` factory reads `marcflavour` syspref
2. Loads the appropriate subclass (`MARC21` or `UNIMARC`)
3. Returns a blessed instance of that subclass
4. The `metadata` accessor lazily loads the MARC record from `$biblio->metadata->record` if only a biblio was passed

## When to use

Use the MetadataExtractor when you need to:
- Extract data from MARC fields that differ between MARC21 and UNIMARC
- Avoid scattering MARC tag numbers across business logic
- Provide a testable, mockable interface for MARC data access

## When NOT to use

- For simple field access where the tag is the same across flavors (e.g., 001)
- When you already have a `MARC::Record` and just need one subfield inline
- For writing/modifying MARC records (this is read-only extraction)

## Adding a new extraction method

1. Add the method to the flavor-specific subclass(es):

```perl
# In Koha::Biblio::Metadata::Extractor::MARC::MARC21
sub get_host_itemnumbers {
    my ($self) = @_;
    my $record = $self->metadata;
    my @itemnumbers;
    for my $field ( $record->field('773') ) {
        my $itemnumber = $field->subfield('9') or next;
        push @itemnumbers, $itemnumber;
    }
    return @itemnumbers;
}

# In Koha::Biblio::Metadata::Extractor::MARC::UNIMARC
sub get_host_itemnumbers {
    my ($self) = @_;
    my $record = $self->metadata;
    my @itemnumbers;
    for my $field ( $record->field('461') ) {
        my $itemnumber = $field->subfield('9') or next;
        push @itemnumbers, $itemnumber;
    }
    return @itemnumbers;
}
```

2. If there's shared logic, put it in the `::MARC` base class.

3. Callers use it without knowing the flavor:

```perl
my $extractor = Koha::Biblio::Metadata::Extractor->new({ biblio => $biblio });
my @host_items = $extractor->get_host_itemnumbers;
```

## Combining with Koha::Object methods

The extractor is a utility — it doesn't replace `Koha::Biblio` methods. The idiomatic pattern is to expose a convenience method on the Koha::Object that uses the extractor internally:

```perl
# In Koha::Biblio
sub host_items {
    my ($self) = @_;
    return Koha::Items->new->empty
        unless C4::Context->preference('EasyAnalyticalRecords');

    my $extractor = Koha::Biblio::Metadata::Extractor->new({ biblio => $self });
    my @itemnumbers = $extractor->get_host_itemnumbers;
    return Koha::Items->new->empty unless @itemnumbers;

    return Koha::Items->search({ itemnumber => { -in => \@itemnumbers } });
}
```

This keeps MARC parsing in the extractor and DB queries in the Koha::Object layer.

## Existing methods

| Method | MARC21 | UNIMARC | Description |
|--------|--------|---------|-------------|
| `get_control_number` | 001 | 001 | Record identifier |
| `get_normalized_upc` | 024$a (ind1=1) | — | UPC barcode |
| `get_normalized_oclc` | 035$a (OCoLC) | — | OCLC number |
| `check_fixed_length` | 005/006/007/008 | — | Validate control field lengths |

## Design decisions

- **Factory pattern**: `Extractor->new` always returns a subclass instance. The base class is never instantiated directly.
- **Lazy metadata loading**: The MARC record is only fetched from the DB when first accessed via `->metadata`.
- **No caching across calls**: Each `->new` creates a fresh instance. Callers that need multiple extractions from the same record should reuse the extractor object.
- **Read-only**: Extractors never modify the MARC record.
