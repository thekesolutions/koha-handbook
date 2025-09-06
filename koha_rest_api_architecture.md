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

This architecture provides Koha with a modern, standards-compliant REST API that seamlessly integrates with the existing Koha Object system while maintaining performance, security, and extensibility through the plugin system.
