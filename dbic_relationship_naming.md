# DBIC Relationship Naming for Koha::Object Accessor Methods

## Rule

Every accessor method on a `Koha::Object` subclass that returns a related resultset **must** have a DBIC relationship with the same name on the corresponding `Koha::Schema::Result` class.

## Why

The `prefetch_whitelist` mechanism checks both:

1. A DBIC relationship with that name exists on the Result class
2. The Koha::Object class has a method with that name (`can($key)`)

If the Koha method exists but the DBIC relationship does not (or has a different name), the embed will work for display but cannot be used for SQL-level sorting, counting, or prefetching. This leads to N+1 queries and prevents `+count` embeds from being sortable.

## Pattern

```perl
# In Koha/Schema/Result/Borrower.pm (below the DO NOT MODIFY line):
__PACKAGE__->has_many(
  "checkouts",
  "Koha::Schema::Result::Issue",
  { "foreign.borrowernumber" => "self.borrowernumber" },
  { cascade_copy => 0, cascade_delete => 0 },
);

# In Koha/Patron.pm:
sub checkouts {
    my ($self) = @_;
    return Koha::Checkouts->_new_from_dbic( scalar $self->_result->checkouts );
}
```

## When the auto-generated relationship has a different name

DBIC schema loader names relationships after the table (e.g. `issues`, `aqbaskets`, `reserves`). When the Koha method uses a cleaner name, add an alias below the `DO NOT MODIFY` line:

```perl
# Table is 'reserves', but Koha method is 'holds'
__PACKAGE__->has_many(
  "holds",
  "Koha::Schema::Result::Reserve",
  { "foreign.biblionumber" => "self.biblionumber" },
  { cascade_copy => 0, cascade_delete => 0 },
);
```

The auto-generated relationship (`reserves`) remains untouched above the line. Both coexist.

## Filtered relationships (coderef conditions)

When the Koha method applies a filter (e.g. `overdues` = checkouts where `date_due < NOW()`), use a DBIC coderef condition:

```perl
__PACKAGE__->has_many(
  "overdues",
  "Koha::Schema::Result::Issue",
  sub {
    my $args = shift;
    return {
      "$args->{foreign_alias}.borrowernumber" => { -ident => "$args->{self_alias}.borrowernumber" },
      "$args->{foreign_alias}.date_due"       => { '<' => \"NOW()" },
    };
  },
  { cascade_copy => 0, cascade_delete => 0 },
);
```

The Koha method then delegates directly:

```perl
sub overdues {
    my ($self) = @_;
    return Koha::Checkouts->_new_from_dbic( scalar $self->_result->overdues );
}
```

This keeps the filter logic in one place and enables both the method and the `+count` sorting infrastructure to use it.

## Checklist for new accessor methods

1. Does a DBIC relationship with the same name exist? If not, add one.
2. Does the Koha method use `$self->_result->relationship_name`? It should.
3. If the method filters results, can the filter be expressed as a DBIC coderef condition? If yes, move it there.
4. If the method involves multi-table joins or syspref-dependent logic that cannot be expressed in DBIC, document it as non-sortable.

## One-liner style

Accessor methods that simply wrap a `has_many` relationship should follow this style:

```perl
sub checkouts {
    my ($self) = @_;
    return Koha::Checkouts->_new_from_dbic( scalar $self->_result->checkouts );
}
```

The `scalar` forces DBIC to return the resultset object rather than a list.

## Historical context

Many older Koha methods (pre-2026) were written before this infrastructure existed. They use the auto-generated relationship name internally (e.g. `$self->_result->issues`) while exposing a cleaner method name (e.g. `checkouts`). Bug 41950 introduced the sortable `+count` embeds infrastructure, which requires the DBIC relationship name to match the Koha method name. Existing methods should be migrated to this pattern as they are touched.

## Annotating unsortable +count embeds (x-koha-unsortable-embeds)

When a `+count` embed is advertised in the API spec but cannot be sorted (because it lacks a matching DBIC relationship or uses complex logic that cannot be expressed as a coderef), it must be annotated with `x-koha-unsortable-embeds` on the list operation.

### Adding the annotation

Add the extension at the operation level (same level as `parameters`, `responses`):

```yaml
/patrons:
  get:
    operationId: listPatrons
    x-koha-unsortable-embeds:
      - overdues+count
    parameters:
      ...
```

This is rendered in the published API docs via the `showExtensions` config in `koha-api-docs`.

### Removing the annotation when sortability is implemented

When a bug resolves the sortability of an embed (by adding a DBIC relationship), remove it from `x-koha-unsortable-embeds`. If it was the last entry, remove the entire extension key from the operation.

Example: after implementing `overdues+count` sorting (bug 42587), the annotation is removed from `GET /patrons` since all `+count` embeds on that endpoint are now sortable.

### Where it lives

- **Koha spec**: `api/v1/swagger/paths/*.yaml` — the `x-koha-unsortable-embeds` key on list operations
- **koha-api-docs**: `config.yaml` — `showExtensions` includes `x-koha-unsortable-embeds`
