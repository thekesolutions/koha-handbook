# Koha API Coding Guidelines

Source: https://wiki.koha-community.org/wiki/Coding_Guidelines_-_API (last synced 2026-05-08)

Supplemental to the general coding guidelines. The `/cities` endpoint (`Koha::REST::V1::Cities`) is the reference implementation.

## REST1: Resources (Swagger Paths)

### REST1.1: Distinct routes

Use distinct paths for distinct operations:

```yaml
"/users": { ... }
"/users/{user_id}": { ... }
```

Don't make path parameters optional to combine routes.

### REST1.2: Resource names

- Use generic, widely-recognized terms (e.g., `/patrons` not `/borrowers`)
- Use plural form for resource names
- Use singular/verbs for action suffixes

### REST1.3: Resource description

All resources defined in their own definition file (`api/v1/swagger/definitions/`).

#### REST1.3.1: type

All fields must have a type.

#### REST1.3.2: required

All resources must specify required fields.

#### REST1.3.3: description

Brief description of each field is strongly recommended.

#### REST1.3.4: mapping

##### REST1.3.4.1: date/datetime/timestamp fields

- Name as `*_date` (not `date_*`)
- Always return full datetime
- Examples: `created_date`, `modified_date`, `submitted_date`, `accepted_date`

##### REST1.3.4.2: action record fields

Format: `action_data`, action in past tense.

##### REST1.3.4.3: relation fields

- FK fields: `related_id` (e.g., `patron_id`, `manager_id`, `creator_id`)
- When embedded, use just the relation name (e.g., `patron` returns a Koha::Patron object)
- `+count` embeds (e.g., `checkouts+count`) require a matching DBIC relationship to be sortable. See [DBIC Relationship Naming](dbic_relationship_naming.md)
- Unsortable `+count` embeds must be annotated with `x-koha-unsortable-embeds` on the list operation

## REST2: HTTP Methods

- **POST** — Create
- **GET** — Read
- **PUT** — Update (full)
- **PATCH** — Update (partial)
- **DELETE** — Delete

For actions that don't map to CRUD on the main object, use action sub-resources:
- `POST /users/{id}/block` — block a user
- `DELETE /users/{id}/block` — unblock

## REST3: Requests and Responses

### REST3.1: Request Parameters

All request parameters must be specified in the OpenAPI spec.

### REST3.2: Response Codes

| Method | Success | Duplicate | Not Found | Conflict |
|--------|---------|-----------|-----------|----------|
| POST   | 201     | 409       | —         | —        |
| GET    | 200     | —         | 404       | —        |
| PUT    | 200     | —         | 404       | —        |
| DELETE  | 204     | —         | 404       | 409      |

### REST3.3: Response Bodies

| Method | Success response |
|--------|-----------------|
| POST   | Created resource representation |
| GET    | Resource representation |
| PUT    | Updated resource representation |
| DELETE  | Empty body |

### REST3.4: Response Headers

POST must include `Location` header pointing to the created resource.

## REST4: Controller Code

### REST4.0: The basics

Every controller starts with:

```perl
my $c = shift->openapi->valid_input or return;
```

All code wrapped in `try/catch` with fallback to `$c->unhandled_exception($_)`:

```perl
return try {
    ...
} catch {
    if ( blessed($_) && ref($_) eq 'Koha::Exceptions::...' ) {
        # handle specific exception
    }
    $c->unhandled_exception($_);
};
```

### REST4.1: Returning Koha::Object(s)

Use the `objects->to_api` helper:

```perl
return $c->render(
    status  => 200,
    openapi => $c->objects->to_api($koha_objects_thing),
);
```

### REST4.2: Finding objects from parameters

```perl
my $patron    = Koha::Patrons->find($c->param('patron_id'));
my $checkouts = $patron->checkouts;
```

### REST4.3: Handling non-existent resources

Use the helper, don't manually return 404:

```perl
my $patron = Koha::Patrons->find($c->param('patron_id'));

return $c->render_resource_not_found('Patron')
    unless $patron;
```

### REST4.4: Deleting a resource

Use the helper:

```perl
$city->delete;
return $c->render_resource_deleted();
```

## Quick Reference: New Endpoint Checklist

1. Define path in `api/v1/swagger/paths/<resource>.yaml`
2. Define schema in `api/v1/swagger/definitions/<resource>.yaml`
3. Register path in `api/v1/swagger/swagger.yaml`
4. Write controller in `Koha/REST/V1/<Resource>.pm`
5. Add `to_api_mapping` in the Koha::Object class
6. Rebuild bundle: `yarn api:bundle`
7. Validate: `prove xt/api.t`
8. Write tests in `t/db_dependent/api/v1/<resource>.t`
