# Koha Database Naming Conventions

## Table names

- Lowercase, snake_case: `library_hours`, `library_weekly_closures`
- Plural for collections: `branches`, `items`, `patrons`
- Junction tables: `{object_type}_branches` (e.g. `itemtypes_branches`)

## Primary key columns

Named `{table_in_singular_form}_id`:

```sql
-- Table: library_weekly_closures → PK: library_weekly_closure_id
-- Table: items → PK: itemnumber (legacy, but new tables use _id)
-- Table: bookings → PK: booking_id
-- Table: return_claims → PK: id (legacy)
```

For new tables, always use `{singular}_id`. Legacy tables may use `id` or domain-specific names (`biblionumber`, `itemnumber`, `borrowernumber`) — don't rename those, but don't follow that pattern for new work.

## Foreign key columns

### Library references

Modern convention: `library_id` (references `branches.branchcode`).

```sql
-- New tables (preferred)
library_id VARCHAR(10) NOT NULL,
FOREIGN KEY (library_id) REFERENCES branches(branchcode)

-- Legacy tables (don't rename, but don't copy the pattern)
branchcode VARCHAR(10) NOT NULL,
FOREIGN KEY (branchcode) REFERENCES branches(branchcode)
```

Reference: `library_hours` uses `library_id`. Older tables like `repeatable_holidays` use `branchcode`.

### Other foreign keys

Name the column after what it references, using `_id` suffix:

```sql
biblio_id      INT  -- references biblios
item_id        INT  -- references items
patron_id      INT  -- references borrowers
item_type_id   VARCHAR(10)  -- references itemtypes
```

## Column types

| Data | Type | Notes |
|------|------|-------|
| Dates | `DATE` | Not separate day/month/year columns |
| Timestamps | `DATETIME` or `TIMESTAMP` | |
| Booleans | `TINYINT(1)` | 0/1, not enums |
| Short text | `VARCHAR(n)` | With appropriate length |
| Long text | `MEDIUMTEXT` or `TEXT` | |
| Money | `DECIMAL(28,6)` | Koha convention for monetary values |

## Constraints

- Always define `FOREIGN KEY` with `ON DELETE CASCADE ON UPDATE CASCADE` for child records
- Add `UNIQUE KEY` constraints where business rules require uniqueness
- Name constraints explicitly when possible for clearer error messages

## API field mapping

The REST API uses snake_case field names that map to database columns. The `to_api_mapping` in `Koha::Object` subclasses handles any renaming:

```perl
# In the Koha::Object subclass
sub to_api_mapping {
    return {
        branchcode => 'library_id',  # legacy column → API name
    };
}
```

For new tables where column names already follow conventions, no mapping is needed.

## Examples

### Legacy (don't follow)
```sql
CREATE TABLE repeatable_holidays (
    id INT AUTO_INCREMENT PRIMARY KEY,     -- generic 'id'
    branchcode VARCHAR(10) NOT NULL,       -- old FK naming
    weekday SMALLINT DEFAULT NULL,         -- overloaded: NULL means different type
    day SMALLINT DEFAULT NULL,
    month SMALLINT DEFAULT NULL,           -- separate day/month instead of DATE
    ...
);
```

### Modern (follow this)
```sql
CREATE TABLE library_weekly_closures (
    library_weekly_closure_id INT AUTO_INCREMENT PRIMARY KEY,
    library_id VARCHAR(10) NOT NULL,
    weekday SMALLINT NOT NULL,
    title VARCHAR(50) NOT NULL DEFAULT '',
    description MEDIUMTEXT NOT NULL,
    FOREIGN KEY (library_id) REFERENCES branches(branchcode)
        ON DELETE CASCADE ON UPDATE CASCADE,
    UNIQUE KEY (library_id, weekday)
);
```
