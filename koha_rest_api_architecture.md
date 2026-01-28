# Koha RESTful API Architecture: Mojolicious and OpenAPI Integration

## Overview

Koha's RESTful API is built on the Mojolicious framework with OpenAPI specification validation. The architecture leverages custom helper plugins that seamlessly integrate with the Koha Object system, providing a powerful and consistent API layer for external integrations and modern web interfaces.

## Core Architecture Components

### 1. Mojolicious Application Structure

The main API application is defined in `Koha::REST::V1`:

```perl
package Koha::REST::V1;
use Mojo::Base 'Mojolicious';

sub startup {
    my $self = shift;
    
    # Load OpenAPI specification
    my $spec_file = $self->home->rel_file("api/v1/swagger/swagger.yaml");
    
    # Register custom helper plugins
    $self->plugin('Koha::REST::Plugin::Objects');
    $self->plugin('Koha::REST::Plugin::Query');
    $self->plugin('Koha::REST::Plugin::Pagination');
    $self->plugin('Koha::REST::Plugin::Exceptions');
    $self->plugin('Koha::REST::Plugin::Responses');
    
    # Configure OpenAPI with authentication
    $self->plugin(OpenAPI => {
        spec  => $spec,
        route => $self->routes->under('/api/v1')->to('Auth#under'),
    });
}
```

### 2. OpenAPI Specification Structure

```
api/v1/
├── swagger/
│   ├── swagger.yaml              # Main API specification
│   ├── definitions/              # Data model definitions
│   │   ├── patron.yaml          # Patron object schema
│   │   ├── biblio.yaml          # Bibliographic record schema
│   │   └── hold.yaml            # Hold object schema
│   └── paths/                   # API endpoint definitions
│       ├── patrons.yaml         # Patron endpoints
│       ├── biblios.yaml         # Biblio endpoints
│       └── holds.yaml           # Hold endpoints
└── app.pl                       # API application entry point
```

### 3. Controller Structure

REST controllers inherit from `Mojolicious::Controller` and use helper plugins:

```perl
package Koha::REST::V1::Holds;
use Mojo::Base 'Mojolicious::Controller';

sub list {
    my $c = shift->openapi->valid_input or return;
    
    return try {
        # Use Koha Objects with helper plugins
        my $holds_set = Koha::Holds->new;
        my $holds = $c->objects->search($holds_set);
        
        return $c->render( status => 200, openapi => $holds );
    } catch {
        $c->unhandled_exception($_);
    };
}
```

## Custom Helper Plugins and Koha Object Integration

### 1. Objects Plugin (`Koha::REST::Plugin::Objects`)

The Objects plugin provides seamless integration between Mojolicious controllers and Koha Objects:

#### Key Helper Methods:

```perl
# Find single object by ID
my $patron = $c->objects->find( Koha::Patrons->new, $patron_id );

# Search with query parameters and pagination
my $patrons = $c->objects->search( Koha::Patrons->new );

# Get resultset for further processing
my $patrons_rs = $c->objects->search_rs( Koha::Patrons->new );

# Find resultset by ID with embeds
my $patron_rs = $c->objects->find_rs( Koha::Patrons->new, $patron_id );
```

#### Integration with Koha Objects:

```perl
# In Koha::REST::Plugin::Objects
$app->helper('objects.search' => sub {
    my ( $c, $result_set, $query_fixers ) = @_;
    
    # Generate resultset using HTTP request information
    my $objects_rs = $c->objects->search_rs( $result_set, $query_fixers );
    
    # Add pagination headers automatically
    $c->add_pagination_headers();
    
    # Convert to API representation
    return $c->objects->to_api($objects_rs);
});
```

### 2. Query Plugin (`Koha::REST::Plugin::Query`)

Handles query parameter parsing and DBIC query generation:

```perl
# Extract reserved parameters (_page, _per_page, _order_by, etc.)
my ( $filtered_params, $reserved_params ) = $c->extract_reserved_params($params);

# Build DBIC query from API parameters
my $query = $c->build_query_params( $filtered_params, $reserved_params );

# Apply query to resultset
my $rs = $result_set->search( $query->{query}, $query->{attributes} );
```

#### Query Parameter Mapping:

```perl
# URL: /api/v1/patrons?surname=Smith&_order_by=firstname&_page=2&_per_page=10
# Generates DBIC query:
{
    query => { surname => 'Smith' },
    attributes => {
        order_by => 'firstname',
        page => 2,
        rows => 10
    }
}
```

### 3. Pagination Plugin (`Koha::REST::Plugin::Pagination`)

Provides automatic pagination support:

```perl
$app->helper('add_pagination_headers' => sub {
    my $c = shift;
    
    # Add standard pagination headers
    $c->res->headers->header('X-Total-Count' => $total_count);
    $c->res->headers->header('X-Base-Total-Count' => $base_total_count);
    
    # Add Link header for navigation
    my @links;
    push @links, qq{<$first_page>; rel="first"} if $first_page;
    push @links, qq{<$prev_page>; rel="prev"} if $prev_page;
    push @links, qq{<$next_page>; rel="next"} if $next_page;
    push @links, qq{<$last_page>; rel="last"} if $last_page;
    
    $c->res->headers->header('Link' => join(', ', @links)) if @links;
});
```

### 4. Exceptions Plugin (`Koha::REST::Plugin::Exceptions`)

Standardizes error handling across the API:

```perl
$app->helper('unhandled_exception' => sub {
    my ( $c, $exception ) = @_;
    
    if ( blessed $exception ) {
        # Handle Koha::Exceptions
        if ( $exception->isa('Koha::Exceptions::Object::NotFound') ) {
            return $c->render( status => 404, openapi => { error => "Object not found" } );
        }
        elsif ( $exception->isa('Koha::Exceptions::BadParameter') ) {
            return $c->render( status => 400, openapi => { error => $exception->parameter } );
        }
    }
    
    # Default server error
    return $c->render( status => 500, openapi => { error => "Internal server error" } );
});
```

## API to Koha Object Mapping

### 1. Automatic Object Conversion

The `to_api()` method in Koha Objects provides automatic conversion:

```perl
# In Koha::Object
sub to_api {
    my ( $self, $params ) = @_;
    
    my $json_object = $self->TO_JSON;
    
    # Apply field mapping from API schema
    my $mapping = $self->api_mapping;
    foreach my $field ( keys %{$mapping} ) {
        my $mapped_field = $mapping->{$field};
        if ( exists $json_object->{$field} ) {
            $json_object->{$mapped_field} = delete $json_object->{$field};
        }
    }
    
    return $json_object;
}
```

### 2. Field Mapping Configuration

Objects define API field mappings:

```perl
# In Koha::Patron
sub api_mapping {
    return {
        borrowernumber => 'patron_id',
        cardnumber     => 'library_id',
        surname        => 'lastname',
        firstname      => 'firstname',
        # ... more mappings
    };
}
```

### 3. Embedded Objects Support

The API supports embedding related objects:

```perl
# URL: /api/v1/patrons/123?_embed=checkouts,holds
# Automatically includes related checkouts and holds in response

# In controller, handled by objects plugin:
my $patron = $c->objects->find_rs( Koha::Patrons->new, $patron_id );
# Plugin automatically applies prefetch based on _embed parameter
```

## OpenAPI Integration Patterns

### 1. Specification-Driven Development

```yaml
# In api/v1/swagger/paths/patrons.yaml
/patrons:
  get:
    x-mojo-to: Patrons#list
    operationId: listPatrons
    parameters:
      - name: surname
        in: query
        type: string
      - name: _page
        in: query
        type: integer
    responses:
      200:
        description: A list of patrons
        schema:
          type: array
          items:
            $ref: "../definitions/patron.yaml"
```

### 2. Automatic Validation

```perl
# OpenAPI plugin automatically validates:
sub add {
    my $c = shift->openapi->valid_input or return;
    # If validation fails, error response is automatically sent
    
    my $body = $c->req->json;  # Validated against schema
    # ... process request
}
```

### 3. Response Formatting

```perl
# Consistent response format
return $c->render(
    status  => 200,
    openapi => $patron->to_api  # Automatically formatted per schema
);
```

## Authentication and Authorization

### 1. Authentication Middleware

```perl
# In Koha::REST::V1::Auth
sub under {
    my $c = shift->openapi->valid_input or return;
    
    # Extract authentication from headers
    my $authorization = $c->req->headers->authorization;
    
    # Validate API key, OAuth token, or session cookie
    my $user = $c->authenticate_api_request($authorization);
    
    return 1 if $user;
    return $c->render( status => 401, openapi => { error => "Authentication required" } );
}
```

### 2. Permission Checking

```perl
# In controller methods
sub delete {
    my $c = shift->openapi->valid_input or return;
    
    # Check permissions using Koha's permission system
    unless ( $c->stash('koha.user')->has_permission({ borrowers => 'delete_borrowers' }) ) {
        return $c->render( status => 403, openapi => { error => "Insufficient permissions" } );
    }
    
    # ... proceed with deletion
}
```

## Plugin Integration

### 1. Plugin Route Registration

```perl
# In Koha::REST::Plugin::PluginRoutes
sub register {
    my ( $self, $app, $config ) = @_;
    
    # Allow plugins to register API routes
    my @plugins = Koha::Plugins->new->GetPlugins({ method => 'api_routes' });
    
    foreach my $plugin (@plugins) {
        my $routes = $plugin->api_routes;
        foreach my $route (@$routes) {
            $app->routes->add_route($route);
        }
    }
}
```

### 2. Plugin API Extension

```perl
# In plugin
sub api_routes {
    my $self = shift;
    
    return [
        {
            spec => {
                '/contrib/myplugin/data' => {
                    'get' => {
                        'x-mojo-to' => 'MyPlugin::Controller#get_data',
                        'operationId' => 'getMyPluginData',
                        # ... OpenAPI specification
                    }
                }
            }
        }
    ];
}
```

## Performance Optimizations

### 1. Query Optimization

```perl
# Automatic prefetch for embedded objects
my $attributes = {};
$c->dbic_merge_prefetch({
    attributes => $attributes,
    result_set => $result_set
});

# Efficient pagination
my $rs = $result_set->search( $query, {
    page => $page,
    rows => $per_page,
    %$attributes
});
```

### 2. Caching Strategies

```perl
# Response caching for expensive operations
$c->res->headers->cache_control('max-age=300') if $cacheable;

# Object caching in Koha::Cache
my $cache_key = "api_patron_$patron_id";
my $cached = Koha::Caches->get_instance->get_from_cache($cache_key);
```

### 3. Lazy Loading

```perl
# Objects plugin supports lazy loading of relationships
# Only loads related data when explicitly requested via _embed
```

## Development Patterns and Best Practices

### 1. Controller Structure

```perl
package Koha::REST::V1::MyResource;
use Mojo::Base 'Mojolicious::Controller';

use Try::Tiny qw( catch try );
use Koha::MyObjects;

sub list {
    my $c = shift->openapi->valid_input or return;
    
    return try {
        my $objects_set = Koha::MyObjects->new;
        my $objects = $c->objects->search($objects_set);
        return $c->render( status => 200, openapi => $objects );
    } catch {
        $c->unhandled_exception($_);
    };
}

sub get {
    my $c = shift->openapi->valid_input or return;
    
    return try {
        my $object = $c->objects->find( Koha::MyObjects->new, $c->param('id') );
        
        unless ($object) {
            return $c->render( status => 404, openapi => { error => "Object not found" } );
        }
        
        return $c->render( status => 200, openapi => $object );
    } catch {
        $c->unhandled_exception($_);
    };
}
```

### 2. Error Handling

```perl
# Consistent error responses
try {
    # ... operation
} catch {
    if ( blessed $_ && $_->isa('Koha::Exceptions::Object::NotFound') ) {
        return $c->render( status => 404, openapi => { error => "Resource not found" } );
    }
    elsif ( blessed $_ && $_->isa('Koha::Exceptions::BadParameter') ) {
        return $c->render( status => 400, openapi => { 
            error => "Bad parameter", 
            parameter => $_->parameter 
        });
    }
    else {
        $c->unhandled_exception($_);
    }
};
```

### 3. Testing Patterns

```perl
# API testing with Test::Mojo
use Test::Mojo;
use t::lib::TestBuilder;

my $t = Test::Mojo->new('Koha::REST::V1');
my $builder = t::lib::TestBuilder->new;

subtest 'GET /api/v1/patrons' => sub {
    my $patron = $builder->build_object({ class => 'Koha::Patrons' });
    
    $t->get_ok('/api/v1/patrons')
      ->status_is(200)
      ->json_has('/0/patron_id')
      ->json_is('/0/firstname' => $patron->firstname);
};
```

## Advanced Features

### 1. Custom Query Filters

```perl
# Support for complex queries
# URL: /api/v1/patrons?q={"surname":{"like":"Sm%"}}
my $query_fixers = [
    sub {
        my ($query, $no_quotes) = @_;
        # Transform API query format to DBIC format
        return $transformed_query;
    }
];

my $objects = $c->objects->search($objects_set, $query_fixers);
```

### 2. Bulk Operations

```perl
# Batch processing support
sub batch_update {
    my $c = shift->openapi->valid_input or return;
    
    my $updates = $c->req->json;
    my @results;
    
    foreach my $update (@$updates) {
        my $object = Koha::MyObjects->find($update->{id});
        $object->set_from_api($update)->store;
        push @results, $object->to_api;
    }
    
    return $c->render( status => 200, openapi => \@results );
}
```

### 3. Streaming Responses

```perl
# Large dataset streaming
sub export {
    my $c = shift->openapi->valid_input or return;
    
    $c->res->headers->content_type('application/json');
    
    my $rs = $c->objects->search_rs(Koha::MyObjects->new);
    
    $c->write('[');
    my $first = 1;
    while (my $object = $rs->next) {
        $c->write(',') unless $first;
        $c->write(encode_json($object->to_api));
        $first = 0;
    }
    $c->write(']');
    $c->finish;
}
```

## API Specification Management

### OpenAPI Bundle Generation

**Critical Requirement**: After making any changes to OpenAPI specification files, you **must** regenerate the API bundle for the changes to take effect.

#### When to Rebuild the Bundle

Rebuild the API bundle whenever you modify:
- `api/v1/swagger/swagger.yaml` (main specification)
- `api/v1/swagger/paths/*.yaml` (endpoint definitions)
- `api/v1/swagger/definitions/*.yaml` (schema definitions)
- `api/v1/swagger/parameters/*.yaml` (parameter definitions)

#### How to Rebuild in KTD

```bash
# In KTD environment (required for dependencies)
ktd --shell --run "cd /kohadevbox/koha && yarn api:bundle"
```

#### What This Does

The `yarn api:bundle` command:
1. Reads the main `swagger.yaml` specification
2. Resolves all `$ref` references to external files
3. Generates `api/v1/swagger/swagger_bundle.json` (gitignored, build artifact)
4. Validates the complete specification for syntax errors

#### Common Issues

**404 Errors in API Tests**: If your new endpoints return 404 in tests, you likely forgot to rebuild the bundle.

**Syntax Errors**: The bundling process will catch YAML syntax errors and invalid OpenAPI specifications.

**Missing Dependencies**: The `redocly` CLI tool is only available in KTD with proper Node.js dependencies installed.

#### Development Workflow

```bash
# 1. Make changes to OpenAPI files
vim api/v1/swagger/paths/my_endpoint.yaml

# 2. Rebuild bundle in KTD
ktd --shell --run "cd /kohadevbox/koha && yarn api:bundle"

# 3. Test your changes
ktd --shell --run "cd /kohadevbox/koha && prove t/db_dependent/api/v1/my_test.t"

# 4. Run QA tools
ktd --shell --run "cd /kohadevbox/koha && /kohadevbox/qa-test-tools/koha-qa.pl -c 2"
```

**Note**: The `swagger_bundle.json` file is automatically generated and should not be manually edited or committed to git.

## Confirmation Flow Pattern

The Koha REST API implements a two-step confirmation flow for operations that require user acknowledgment of warnings or special conditions. This pattern is used for checkouts and other operations where the system needs to inform the user about potential issues before proceeding.

### Overview

The confirmation flow uses JWT tokens to securely encode the exact conditions that existed at the time of the availability check, preventing race conditions and replay attacks.

### Flow Diagram

```
Client                          API Server
  |                                |
  |  GET /checkouts/availability   |
  |------------------------------->|
  |                                | Check availability
  |                                | Generate JWT token
  |  200 OK + confirmation_token   |
  |<-------------------------------|
  |                                |
  | User reviews confirmations     |
  |                                |
  |  POST /checkouts               |
  |  + confirmation token          |
  |------------------------------->|
  |                                | Validate token
  |                                | Perform checkout
  |  201 Created                   |
  |<-------------------------------|
```

### Step 1: Check Availability

**Endpoint**: `GET /api/v1/checkouts/availability`

**Parameters**:
- `patron_id` - The patron attempting to checkout
- `item_id` - The item to checkout

**Response** (200 OK):
```json
{
  "blockers": {},
  "confirms": {
    "RESERVED": {
      "patron_id": 456,
      "patron_name": "Jane Doe"
    },
    "TOO_MANY": 5
  },
  "warnings": {
    "DEBT": "10.50"
  },
  "confirmation_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response Categories**:

- **blockers**: Conditions that prevent the operation (returns 403 if present)
  - Examples: `DEBARRED`, `EXPIRED`, `CARD_LOST`, `ITEM_LOST`
  
- **confirms**: Conditions requiring user confirmation
  - Examples: `RESERVED` (item on hold for another patron), `TOO_MANY` (patron has many checkouts), `DEBT` (patron has fines)
  
- **warnings**: Informational messages that don't prevent the operation
  - Examples: `AGE_RESTRICTION`, `ADDITIONAL_MATERIALS`

**Token Generation**:
```perl
# Controller generates token from confirmation keys
my @confirm_keys = sort keys %{$confirmation};
unshift @confirm_keys, $item->id;
unshift @confirm_keys, $user->id;

my $token = Koha::Token->new->generate_jwt({ 
    id => join(':', @confirm_keys) 
});
```

### Step 2: Perform Operation with Confirmation

**Endpoint**: `POST /api/v1/checkouts`

**Body**:
```json
{
  "patron_id": 123,
  "item_id": 789,
  "confirmation": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Token Validation**:
```perl
if (keys %{$confirmation}) {
    my $confirmed = 0;
    
    if (my $token = $c->param('confirmation')) {
        # Rebuild the same key string
        my $confirm_keys = join(":", sort keys %{$confirmation});
        $confirm_keys = $user->id . ":" . $item->id . ":" . $confirm_keys;
        
        # Verify JWT matches
        $confirmed = Koha::Token->new->check_jwt({ 
            id => $confirm_keys, 
            token => $token 
        });
    }
    
    unless ($confirmed) {
        return $c->render(
            status => 412,
            openapi => { error => "Confirmation error" }
        );
    }
}
```

**Response** (201 Created):
```json
{
  "checkout_id": 12345,
  "patron_id": 123,
  "item_id": 789,
  "due_date": "2026-02-15T23:59:59Z",
  ...
}
```

### HTTP Status Codes

- **200 OK**: Availability checked successfully
- **201 Created**: Operation completed successfully
- **403 Forbidden**: Blockers prevent the operation
- **412 Precondition Failed**: Confirmations required but token missing/invalid
- **409 Conflict**: Resource not found (patron/item)

### Security Considerations

1. **Token Binding**: The JWT token includes:
   - User ID (who is performing the operation)
   - Item ID (what is being operated on)
   - Confirmation keys (what conditions existed)

2. **Replay Prevention**: The token is only valid for the specific combination of user, item, and conditions

3. **Race Condition Protection**: If conditions change between availability check and operation, the token validation will fail

4. **Token Expiration**: JWT tokens have a built-in expiration time

### Implementation Example

```perl
# In controller
sub get_availability {
    my $c = shift->openapi->valid_input or return;
    
    my $patron = Koha::Patrons->find($c->param('patron_id'));
    my $item = Koha::Items->find($c->param('item_id'));
    
    # Use availability class
    my $result = $item->checkout_availability({ patron => $patron });
    
    # Generate token from confirmation keys
    my @confirm_keys = sort keys %{$result->confirmations};
    unshift @confirm_keys, $item->id;
    unshift @confirm_keys, $c->stash('koha.user')->id;
    
    my $token = Koha::Token->new->generate_jwt({ 
        id => join(':', @confirm_keys) 
    });
    
    return $c->render(
        status => 200,
        openapi => {
            blockers => $result->blockers,
            confirms => $result->confirmations,
            warnings => $result->warnings,
            confirmation_token => $token
        }
    );
}
```

### Client Implementation Guidelines

1. **Always check availability first** before attempting the operation
2. **Display confirmations to the user** with clear explanations
3. **Include the token** when user confirms they want to proceed
4. **Handle 412 responses** by re-checking availability (conditions may have changed)
5. **Don't cache tokens** - they are single-use and time-limited

### Public vs Staff Interface

For public (OPAC) interfaces, some confirmations are upgraded to blockers:

```perl
if ($c->stash('is_public')) {
    # Upgrade confirmations to blockers for public interface
    my @should_block = qw/TOO_MANY ISSUED_TO_ANOTHER RESERVED 
                          RESERVE_WAITING TRANSFERRED PROCESSING 
                          AGE_RESTRICTION/;
    
    for my $block (@should_block) {
        if (exists($confirmation->{$block})) {
            $impossible->{$block} = $confirmation->{$block};
            delete $confirmation->{$block};
        }
    }
}
```

This ensures patrons cannot override certain restrictions that require staff intervention.

## API Specification Management

This architecture provides Koha with a modern, standards-compliant REST API that seamlessly integrates with the existing Koha Object system while maintaining performance, security, and extensibility through the plugin system.
