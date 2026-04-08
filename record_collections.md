# Record Collections: MARC Serialization for the REST API

## The problem

Koha's REST API needs to return lists of bibliographic and authority records in multiple MARC formats (MARCXML, MARC-in-JSON, raw USMARC, plain text). Each format has different collection wrappers, separators, and serialization functions. This logic shouldn't live in every controller.

## The solution: `Koha::Objects::Record::Collections`

A mixin class that any `Koha::Objects` subclass can inherit to gain `print_collection()` — a single method that serializes the entire result set into any supported MARC format.

### Inheritance

```perl
# Koha::Biblios gets collection serialization by mixing in the role
package Koha::Biblios;
use base qw(Koha::Objects Koha::Objects::Record::Collections);
```

Current consumers:
- `Koha::Biblios` — live bibliographic records
- `Koha::Old::Biblios` — deleted bibliographic records
- `Koha::Authorities` — authority records

### The `print_collection` method

```perl
my $marcxml = $result_set->print_collection({ format => 'marcxml' });
my $mij     = $result_set->print_collection({ format => 'mij' });
my $usmarc  = $result_set->print_collection({ format => 'marc' });
my $text    = $result_set->print_collection({ format => 'txt' });

# With item data embedded into the MARC records (952 fields)
my $marcxml = $result_set->print_collection({ format => 'marcxml', embed_items => 1 });
```

It iterates the result set, serializes each record, and wraps them in the appropriate collection structure:

| Format | Start | Glue | End | Serializer |
|--------|-------|------|-----|------------|
| `marcxml` | XML collection header | (none) | XML collection footer | `MARC::File::XML::record` |
| `mij` | `[` | `,` | `]` | `MARC::File::MiJ::encode` |
| `marc` | (none) | `\n` | (none) | `MARC::File::USMARC::encode` |
| `txt` | (none) | `\n\n` | (none) | `MARC::Record::as_formatted` |

### How `embed_items` works

Each element in the result set is a `Koha::Biblio` (or similar). The method chooses how to get the MARC record:

```perl
my $record = $embed_items
    ? $element->metadata_record({ embed_items => 1 })
    : $element->record;
```

- `$element->record` — returns the raw MARC from `biblio_metadata`, no items
- `$element->metadata_record({ embed_items => 1 })` — runs the `EmbedItems` RecordProcessor filter, which loads all items from the DB and injects them as 952 datafields into the MARC record

This is the same mechanism Koha uses internally (e.g. XSLT views, OAI-PMH). The REST API just exposes it via the `x-koha-embed: items` header.

### How the controller uses it

In `Koha::REST::V1::Biblios::list`:

```perl
my $embed       = $c->stash('koha.embed');
my $embed_items = exists $embed->{items} ? 1 : 0;

if ( $c->req->headers->accept =~ m/application\/marcxml\+xml/ ) {
    $c->res->headers->add( 'Content-Type', 'application/marcxml+xml' );
    return $c->render(
        status => 200,
        text   => $biblios->print_collection({ format => 'marcxml', embed_items => $embed_items })
    );
}
```

The `koha.embed` stash is populated by Koha's middleware when the client sends `x-koha-embed: items`. The controller just passes the flag through.

## Design principles to follow

### 1. The collection class handles serialization, not the controller

Controllers should never build MARC XML strings or JSON arrays manually. Call `print_collection` and let it handle format differences.

### 2. The element contract

Each element in the result set must provide:
- `->record` — returns a `MARC::Record` (the raw stored record)
- `->metadata_record({ embed_items => 1 })` — returns a `MARC::Record` with items embedded (optional, only needed if `embed_items` support is desired)
- `->record_schema` (optional) — returns the MARC schema (e.g. `MARC21`, `UNIMARC`). Falls back to the `marcflavour` system preference.

### 3. Adding a new format

Add the serializer to the `%serializers` hash and define its `$start`, `$glue`, `$end` in the format dispatch block. No controller changes needed.

### 4. Adding a new embeddable option

Follow the `embed_items` pattern:
1. Add the parameter to `print_collection`'s hashref
2. Use it to choose how to obtain the record from each element
3. Declare it in the swagger spec's `x-koha-embed` enum for the endpoint
4. The controller reads it from `$c->stash('koha.embed')` and passes it through

### 5. Parameters must use a hashref

Per Koha coding guidelines, methods with more than one parameter use a hashref:

```perl
# Correct
$rs->print_collection({ format => 'marcxml', embed_items => 1 });

# Wrong — positional params
$rs->print_collection('marcxml', 1);
```

## When to use this pattern

Use `Koha::Objects::Record::Collections` when:
- You have a `Koha::Objects` subclass whose elements represent MARC-based records
- You need to serialize a list of those records for an API response
- You want consistent multi-format support without duplicating serialization logic

Don't use it for:
- Single-record responses (use `$biblio->metadata->record` directly)
- Non-MARC data (JSON-only endpoints use the standard `objects->search` → `render` pattern)
