# Elasticsearch dynamic mapping date detection bug

## Problem

When `ElasticsearchMARCFormat` is set to `ARRAY`, Koha stores the full MARC
record as a structured JSON object in the `marc_data_array` field. This field
is configured with `dynamic: true` in `field_config.yaml`, which means
Elasticsearch auto-detects field types based on the first document indexed.

If a MARC subfield's first value looks like a date (e.g. `2024-01-15`),
ES permanently locks that subfield's mapping to type `date`. All subsequent
records with non-date values in the same subfield **fail to index** with a
`document_parsing_exception`.

This affects any subfield stored in `marc_data_array` that is not explicitly
mapped in Koha's search field configuration — most notably item subfields
like 952$e (acquisition source), 952$d (date of acquisition), and 952$w
(replacement price date).

## Root cause

`field_config.yaml` configures `marc_data_array` as:

```yaml
marc_data_array:
  type: object
  dynamic: true
```

There is no `date_detection: false`, so ES uses its default behavior of
guessing that strings matching date patterns should be mapped as `date` type.

## Reproduction steps

```bash
# 1. Launch a fresh KTD
$ ktd --search-engine es8 --name broken-es --proxy up -d
$ ktd --name broken-es --wait-ready 180

# 2. Get a shell
$ ktd --name broken-es --shell
```

```bash
# 3. Set ARRAY format, flush cache, wipe records, reset index
k$ koha-mysql kohadev -e "UPDATE systempreferences SET value='ARRAY' \
     WHERE variable='ElasticsearchMARCFormat';"
k$ echo 'flush_all' | nc -q1 memcached 11211
k$ koha-mysql kohadev -e "DELETE FROM biblio_metadata; \
     DELETE FROM items; DELETE FROM biblioitems; DELETE FROM biblio;"
k$ koha-shell kohadev -c \
     '/usr/share/koha/bin/search_tools/rebuild_elasticsearch.pl -d -r --biblios'
```

```bash
# 4. Add a record with a date-like value in 952$e (booksellerid)
k$ koha-shell kohadev -p -c 'perl -MC4::Biblio -MMARC::Record -MMARC::Field \
     -MKoha::Item -e "
my \$r = MARC::Record->new();
\$r->append_fields(
    MARC::Field->new(q{245}, q{1}, q{0}, a => q{Record one}),
    MARC::Field->new(q{942}, q{ }, q{ }, c => q{BK}),
);
my (\$bn) = C4::Biblio::AddBiblio(\$r, q{});
Koha::Item->new({
    biblionumber     => \$bn,
    biblioitemnumber => \$bn,
    homebranch       => q{CPL},
    holdingbranch    => q{CPL},
    itype            => q{BK},
    booksellerid     => q{2024-01-15},
})->store;
print qq{Added biblio \$bn\n};
"'
```

```bash
# 5. Index it — this "poisons" the mapping
k$ koha-shell kohadev -c \
     '/usr/share/koha/bin/search_tools/rebuild_elasticsearch.pl --biblios -bn 441'
```

```bash
# 6. Verify ES auto-detected 952$e as type "date"
k$ curl -s 'es:9200/koha_kohadev_biblios/_mapping/field/marc_data_array.fields.952.subfields.e'
# Expected: {"koha_kohadev_biblios":{"mappings":{"marc_data_array.fields.952.subfields.e":{"full_name":"marc_data_array.fields.952.subfields.e","mapping":{"e":{"type":"date"}}}}}}
```

```bash
# 7. Add a second record with a non-date value in 952$e
k$ koha-shell kohadev -p -c 'perl -MC4::Biblio -MMARC::Record -MMARC::Field \
     -MKoha::Item -e "
my \$r = MARC::Record->new();
\$r->append_fields(
    MARC::Field->new(q{245}, q{1}, q{0}, a => q{Record two}),
    MARC::Field->new(q{942}, q{ }, q{ }, c => q{BK}),
);
my (\$bn) = C4::Biblio::AddBiblio(\$r, q{});
Koha::Item->new({
    biblionumber     => \$bn,
    biblioitemnumber => \$bn,
    homebranch       => q{CPL},
    holdingbranch    => q{CPL},
    itype            => q{BK},
    booksellerid     => q{My Vendor Name},
})->store;
print qq{Added biblio \$bn\n};
"'
```

```bash
# 8. Try to index — THIS FAILS
k$ koha-shell kohadev -c \
     '/usr/share/koha/bin/search_tools/rebuild_elasticsearch.pl --biblios -bn 442 -v -v -v'
# Expected error:
# Record #442 failed to parse field [marc_data_array.fields.952.subfields.e]
# of type [date] in document with id '442'. Preview of field's value:
# 'My Vendor Name' (document_parsing_exception) :
# illegal_argument_exception (failed to parse date field [My Vendor Name]
# with format [strict_date_optional_time||epoch_millis])
```

```bash
# Cleanup
$ ktd --name broken-es down
```

## Possible fixes

1. **Add `date_detection: false`** to `marc_data_array` in `field_config.yaml`.
   Koha does not currently pass this parameter through to ES when creating the
   index mapping.

2. **Set `dynamic: false`** on `marc_data_array`. This prevents auto-mapping
   entirely, but breaks whole-record search (`marc_data_array.*` in
   `QueryBuilder.pm`).

3. **Use a dynamic template** to force all `marc_data_array` subfields to
   `text` type. Not currently supported by `field_config.yaml`.
