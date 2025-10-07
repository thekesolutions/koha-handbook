# Koha Search Architecture

## Overview

Koha uses Elasticsearch as its primary search engine, with a sophisticated mapping system that controls which MARC fields are searchable and how they're indexed.

## ElasticsearchMARCFormat System Preference

Controls how MARC records are stored in Elasticsearch:

### `base64ISO2709` (Default)
- Stores MARC as base64-encoded ISO2709 format
- **Pros**: Faster indexing, smaller storage footprint
- **Cons**: MARC data not searchable within Elasticsearch
- **Storage**: `marc_data` field contains encoded binary data

### `ARRAY`
- Stores MARC as searchable JSON array structure
- **Pros**: Full MARC structure searchable, better for debugging
- **Cons**: Slower indexing, larger storage requirements
- **Storage**: `marc_data_array` field contains structured JSON

**Example ARRAY format:**
```json
{
  "marc_format": "ARRAY",
  "marc_data_array": {
    "leader": "01344cam a22003014i 4500",
    "fields": [
      {"001": "17446121"},
      {"245": {
        "ind1": "1", "ind2": "0",
        "subfields": [
          {"a": "Title"},
          {"b": "subtitle"}
        ]
      }}
    ]
  }
}
```

## Search Field Mapping System

### Database Tables

**`search_field`**: Defines searchable fields
- `name`: Internal field name (e.g., "title", "author")
- `label`: Human-readable label
- `staff_client`, `opac`: Visibility flags

**`search_marc_map`**: Maps MARC fields to search concepts
- `marc_field`: MARC field (e.g., "245a", "100a")
- `index_name`: "biblios" or "authorities"

**`search_marc_to_field`**: Links MARC mappings to search fields
- Connects `search_marc_map` entries to `search_field` entries

### Default Behavior

By default, Koha searches **only explicitly mapped fields** (~152 fields). Unmapped MARC fields like 856 are **not searchable** through the standard interface.

## Search Query Types

### Standard Search
```perl
use Koha::SearchEngine::Elasticsearch::Search;
my $searcher = Koha::SearchEngine::Elasticsearch::Search->new({index => "biblios"});
my ($error, $results) = $searcher->simple_search_compat("query", 0, 20);
```
- Searches only mapped fields
- Fast and efficient
- Limited to configured field mappings

### Whole Record Search (`whole_record=on`)
```perl
my ($error, $query, ...) = $builder->build_query_compat(
    [], ["search term"], ["kw,phr"], [], [], 0, "en", 
    { whole_record => 1 }
);
```
- **Key difference**: Adds `marc_data_array.*` to search fields
- Searches entire MARC record structure
- Required for finding content in unmapped fields (like 856)
- Available in advanced search interface

### Code Implementation
```perl
# From QueryBuilder.pm line 203-205
if ( $options{whole_record} ) {
    push @$fields, 'marc_data_array.*';
}
```

## 856 Field Searchability

### Problem
856 fields (Electronic Location and Access) are **not mapped** by default:
- Content stored in `marc_data_array` but not indexed as searchable fields
- Standard searches cannot find 856 content
- Users cannot discover electronic resources through normal search

### Solution Options

**Option 1: Add 856 Field Mappings**
```sql
-- Add search field
INSERT INTO search_field (name, label, type, staff_client, opac) 
VALUES ("electronic-resource", "Electronic resource", "", 1, 1);

-- Add MARC mappings
INSERT INTO search_marc_map (index_name, marc_type, marc_field) VALUES 
("biblios", "marc21", "856u"),
("biblios", "marc21", "856y"), 
("biblios", "marc21", "856z");

-- Link mappings to field
INSERT INTO search_marc_to_field (search_marc_map_id, search_field_id)
SELECT smm.id, sf.id FROM search_marc_map smm, search_field sf 
WHERE smm.marc_field IN ("856u", "856y", "856z") AND sf.name = "electronic-resource";

-- Rebuild index (without --reset to preserve mappings)
koha-elasticsearch --rebuild -d -b kohadev
```

**Option 2: Use Whole Record Search**
- Enable "Search entire record" in advanced search
- URL parameter: `whole_record=on`
- Searches `marc_data_array.*` including 856 fields

## Query Structure Analysis

### Standard Query (152 fields)
```json
{
  "query_string": {
    "fields": ["title", "author", "subject", ...], // 152 mapped fields
    "query": "search term",
    "default_operator": "AND"
  }
}
```

### Whole Record Query (153 fields)
```json
{
  "query_string": {
    "fields": ["title", "author", "subject", ..., "marc_data_array.*"],
    "query": "search term", 
    "default_operator": "AND"
  }
}
```

## Performance Considerations

### ElasticsearchMARCFormat Impact
- **base64ISO2709**: Whole record search impossible (binary data)
- **ARRAY**: Whole record search works but may hit query complexity limits

### Query Complexity
- Whole record searches can trigger "too many nested clauses" errors
- Elasticsearch `maxClauseCount` limit (default: 1724)
- Consider simpler queries or field-specific mappings for production

## Best Practices

### For Administrators
1. **Choose MARC format based on needs**:
   - Use `ARRAY` if whole record search is required
   - Use `base64ISO2709` for performance in large catalogs

2. **Map important fields explicitly**:
   - Don't rely on whole record search for frequently used fields
   - Create specific mappings for 856, 5XX notes, etc.

3. **Test search behavior**:
   - Verify field mappings after changes
   - Use `--rebuild` (not `--reset`) to preserve custom mappings

### For Developers
1. **Understanding search scope**:
   ```perl
   # Check what fields are being searched
   my $builder = Koha::SearchEngine::Elasticsearch::QueryBuilder->new({index => "biblios"});
   my $query = $builder->build_query("test");
   my $fields = $query->{query}->{bool}->{must}->[0]->{query_string}->{fields};
   print "Searching ", scalar(@$fields), " fields\n";
   ```

2. **Reproduce UI searches**:
   ```perl
   # Standard search: /catalogue/search.pl?q=term
   my ($error, $results) = $searcher->simple_search_compat("term", 0, 20);
   
   # Advanced search with whole record: ?advsearch=1&whole_record=on
   my ($error, $query, ...) = $builder->build_query_compat(
       [], ["term"], ["kw,phr"], [], [], 0, "en", { whole_record => 1 }
   );
   ```

## Troubleshooting

### 856 Fields Not Searchable
- **Symptom**: Electronic resource links not found in search
- **Cause**: No field mappings for 856
- **Solutions**: Add mappings OR use whole record search

### Search Returns No Results
- **Check**: Field mappings exist for the MARC fields
- **Check**: ElasticsearchMARCFormat setting
- **Check**: Index rebuild completed successfully

### "Too Many Nested Clauses" Error
- **Cause**: Complex queries with whole record search
- **Solutions**: Simplify query OR add specific field mappings

## Related System Preferences

- `ElasticsearchMARCFormat`: Controls MARC storage format
- `ElasticsearchCrossFields`: Enables cross-field matching
- `ElasticsearchPreventAutoTruncate`: Controls automatic truncation
- `numSearchResults`: Default results per page

## Architecture Summary

Koha's search architecture balances performance and functionality through:
1. **Selective field mapping** for fast, targeted searches
2. **Whole record search** for comprehensive coverage
3. **Flexible MARC storage** formats for different use cases
4. **Configurable field mappings** for institutional customization

Understanding this architecture is crucial for optimizing search performance and ensuring all content is discoverable by users.
