# Koha plugin architecture

Comprehensive guide to Koha plugin development, covering the plugin framework, data storage, lifecycle management, and architectural patterns.

## Plugin framework overview

Koha plugins extend the core functionality through a standardized framework built on `Koha::Plugins::Base`. The framework provides:

- **Lifecycle Management**: Install, upgrade, uninstall hooks
- **Data Persistence**: Plugin-specific database storage
- **Template Integration**: Custom templates and UI components
- **Hook System**: Integration points with core Koha functionality
- **Configuration Management**: Persistent settings storage

## Plugin data storage methods

### Core storage API

Koha plugins inherit from `Koha::Plugins::Base` which provides persistent data storage methods:

```perl
# Store data in plugin database table
$self->store_data({ key => 'value', another_key => 'another_value' });

# Retrieve data from plugin database table
my $value = $self->retrieve_data('key');
my $all_data = $self->retrieve_data();  # Returns hashref of all stored data

# Common patterns
my $config_yaml = $self->retrieve_data('configuration') || '';
my $cached_data = $self->retrieve_data('cached_configuration');

# Clear cached data
$self->store_data({ cached_configuration => undef });
```

### Data storage characteristics

- **Persistent**: Data survives plugin upgrades and Koha restarts
- **Plugin-specific**: Each plugin has its own data namespace
- **Key-value**: Simple hash-based storage system
- **Serialization**: Complex data structures are automatically serialized
- **Database-backed**: Stored in `plugin_data` table in Koha database

### Configuration management pattern

**Standard Configuration Loading:**
```perl
sub configuration {
    my ($self) = @_;
    
    # Check for cached configuration first
    my $cached_config = $self->retrieve_data('cached_configuration');
    return $cached_config if $cached_config;
    
    # Load raw YAML configuration
    my $config_yaml = $self->retrieve_data('configuration') || '';
    return {} unless $config_yaml;
    
    # Parse and process configuration
    my $config = try {
        YAML::XS::Load($config_yaml);
    } catch {
        warn "Invalid YAML configuration: $_";
        return {};
    };
    
    # Apply defaults and transformations
    $self->_process_configuration($config);
    
    # Cache processed configuration
    $self->store_data({ cached_configuration => $config });
    
    return $config;
}
```

**Cache Invalidation:**
```perl
# Force configuration reload
$self->store_data({ cached_configuration => undef });
my $fresh_config = $self->configuration();
```

**Configuration with Defaults:**
```perl
sub _process_configuration {
    my ($self, $config) = @_;
    
    foreach my $section_name (keys %$config) {
        my $section = $config->{$section_name};
        
        # Apply defaults
        $section->{enabled} //= 1;
        $section->{timeout} //= 30;
        $section->{max_retries} //= 3;
        
        # Transform data structures
        if ($section->{library_mappings}) {
            $self->_transform_library_mappings($section->{library_mappings});
        }
    }
}
```

## Plugin lifecycle methods

### Essential plugin methods

```perl
package Koha::Plugin::Com::Company::PluginName;
use base qw(Koha::Plugins::Base);

sub new {
    my ($class, $args) = @_;
    $args->{'metadata'} = {
        name            => 'Plugin Name',
        version         => '1.0.0',
        minimum_version => '22.11.00.000',
        description     => 'Plugin description',
        author          => 'Author Name',
    };
    return $class->SUPER::new($args);
}

sub install {
    my ($self) = @_;
    
    # Create database tables
    my $dbh = C4::Context->dbh;
    $dbh->do(q{
        CREATE TABLE IF NOT EXISTS plugin_table (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    });
    
    # Set default configuration
    $self->store_data({
        configuration => $self->_default_configuration_yaml(),
        version_installed => $self->get_metadata()->{version}
    });
}

sub upgrade {
    my ($self, $args) = @_;
    
    my $dt = dt_from_string();
    $self->store_data({ 
        last_upgraded => $dt->ymd('-') . ' ' . $dt->hms(':'),
        version_upgraded_from => $args->{version}
    });
    
    # Perform version-specific upgrades
    my $current_version = $args->{version};
    if (version->parse($current_version) < version->parse('2.0.0')) {
        $self->_upgrade_to_v2();
    }
}

sub uninstall {
    my ($self) = @_;
    
    # Drop plugin tables
    my $dbh = C4::Context->dbh;
    $dbh->do("DROP TABLE IF EXISTS plugin_table");
    
    # Plugin data is automatically cleaned up by framework
}
```

### Version management

**Version Comparison:**
```perl
use version;

sub needs_upgrade {
    my ($self) = @_;
    
    my $installed_version = $self->retrieve_data('version_installed') || '0.0.0';
    my $current_version = $self->get_metadata()->{version};
    
    return version->parse($current_version) > version->parse($installed_version);
}
```

## Configuration validation

### Validation framework

```perl
sub check_configuration {
    my ($self) = @_;
    
    my $config = $self->configuration();
    my @errors;
    
    # Validate each configuration section
    foreach my $section_name (keys %$config) {
        my $section = $config->{$section_name};
        
        # Required fields validation
        push @errors, $self->_validate_required_fields($section_name, $section);
        
        # Data type validation
        push @errors, $self->_validate_data_types($section_name, $section);
        
        # Business logic validation
        push @errors, $self->_validate_business_rules($section_name, $section);
    }
    
    return \@errors;
}

sub _validate_required_fields {
    my ($self, $section_name, $section) = @_;
    my @errors;
    
    my @required_fields = qw(api_base_url client_id client_secret);
    
    foreach my $field (@required_fields) {
        unless ($section->{$field}) {
            push @errors, "Section '$section_name' is missing required field '$field'";
        }
    }
    
    return @errors;
}
```

### Configuration scripts

**Generic Configuration Management Script Pattern:**
```perl
#!/usr/bin/env perl

use Modern::Perl;
use Getopt::Long;
use YAML::XS;
use Try::Tiny qw(catch try);

my ($dump, $load, $file, $force, $help);

GetOptions(
    'dump'   => \$dump,
    'load'   => \$load,
    'file=s' => \$file,
    'force'  => \$force,
    'help'   => \$help,
);

my $plugin = Koha::Plugin::Com::Company::PluginName->new;

if ($dump) {
    my $config_yaml = $plugin->retrieve_data('configuration') || '';
    
    if ($file) {
        open my $fh, '>', $file or die "Cannot open '$file': $!";
        print $fh $config_yaml;
        close $fh;
    } else {
        print $config_yaml;
    }
}

if ($load) {
    my $yaml_content = $file ? do { 
        open my $fh, '<', $file or die "Cannot open '$file': $!";
        local $/; <$fh>
    } : do {
        local $/; <STDIN>
    };
    
    # Validate YAML syntax
    try {
        YAML::XS::Load($yaml_content);
    } catch {
        die "Invalid YAML syntax: $_";
    };
    
    # Store and validate
    my $original_config = $plugin->retrieve_data('configuration');
    $plugin->store_data({ configuration => $yaml_content });
    $plugin->store_data({ cached_configuration => undef });
    
    my $errors = $plugin->check_configuration();
    
    if (@$errors && !$force) {
        # Restore original configuration
        $plugin->store_data({ configuration => $original_config });
        $plugin->store_data({ cached_configuration => undef });
        
        say STDERR "Configuration validation failed:";
        say STDERR "  - $_" for @$errors;
        exit 1;
    }
    
    if (@$errors && $force) {
        say STDERR "Configuration warnings (forced):";
        say STDERR "  - $_" for @$errors;
    }
}
```

## Plugin database integration

### Custom schema classes

**Plugin-Specific Objects:**
```perl
# Plugin object class
package Koha::Plugin::Com::Company::PluginName::MyRecord;
use base qw(Koha::Object);

sub _type {
    return 'PluginMyrecord';  # Must match schema class name
}

# Plugin collection class
package Koha::Plugin::Com::Company::PluginName::MyRecords;
use base qw(Koha::Objects);

sub _type {
    return 'Koha::Plugin::Com::Company::PluginName::MyRecord';
}
```

**Schema Registration:**
```perl
# In plugin's main class BEGIN block
BEGIN {
    push @INC, dirname(__FILE__) . '/lib';
}

# Register plugin schema classes
use Koha::Schema;
Koha::Schema->register_class('PluginMyrecord', 'Koha::Schema::Result::PluginMyrecord');
```

### Database table management

**Table Creation in install():**
```perl
sub install {
    my ($self) = @_;
    
    my $dbh = C4::Context->dbh;
    
    # Create plugin tables
    $dbh->do(q{
        CREATE TABLE IF NOT EXISTS plugin_records (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            status ENUM('pending', 'processed', 'failed') DEFAULT 'pending',
            data JSON,
            created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            INDEX idx_status (status),
            INDEX idx_created (created_on)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
    });
}
```

## Template integration

### Plugin templates

**Template Directory Structure:**
```
Koha/Plugin/Com/Company/PluginName/
├── templates/
│   ├── intranet/
│   │   ├── configuration.tt
│   │   └── reports.tt
│   └── opac/
│       └── public_interface.tt
```

**Template Loading:**
```perl
sub tool {
    my ($self, $args) = @_;
    
    my $template = $self->get_template({ file => 'configuration.tt' });
    
    $template->param(
        configuration => $self->configuration(),
        errors => $self->check_configuration(),
    );
    
    $self->output_html($template->output());
}
```

**Template Wrapper:**
Koha provides a template wrapper for plugins that will automatically make the breadcrumbs, set the correct search bar, and set the correct aside. 

```perl
[% WRAPPER "wrapper-staff-tool-plugin.inc" method="{configure||tool||report}" plugin_title="{Plugin Title}" %]
#Content
[% END %]<!--end wrapper-->
```

## Hook system integration

### Available hooks

**Common Plugin Hooks:**
```perl
# Modify notice content
sub notices_content {
    my ($self, $args) = @_;
    
    my $notice = $args->{notice};
    my $content = $args->{content};
    
    # Add plugin-specific content
    $content->{plugin_data} = $self->_get_notice_data($notice);
    
    return $content;
}

# Add JavaScript/CSS to pages
sub intranet_js {
    my ($self) = @_;
    return $self->get_template({ file => 'intranet.js' })->output();
}

# Modify catalog search results
sub opac_results_xslt_variables {
    my ($self, $args) = @_;
    
    my $variables = $args->{variables};
    $variables->{plugin_enhanced} = 1;
    
    return $variables;
}
```

## Avoiding "Subroutine redefined" Warnings

### The problem

Plugins that ship their own library classes (e.g. `Koha::Object` subclasses, API clients, converters) under a `lib/` directory commonly hit "Subroutine redefined" warnings during `install_plugins.pl` or Plack startup. This happens when the same module is loaded twice via different `@INC` paths — for example, the plugin's `BEGIN` block adds `lib/` to `@INC` and eagerly `require`s classes, then the Controller `use`s them again through a path that resolves differently (e.g. symlinks in development environments).

**Symptoms:**
```
Subroutine new redefined at .../lib/MyPlugin/Client.pm line 41.
Subroutine process redefined at .../lib/MyPlugin/Converter.pm line 28.
```

### The solution: factory methods with lazy loading

Instead of having the Controller (or other consumers) directly `use` the library classes, the plugin class provides factory methods that `require` the class on first call. The Controller only `use`s the plugin class itself.

**Plugin class — factory methods:**
```perl
package Koha::Plugin::Com::Company::MyPlugin;

use base qw(Koha::Plugins::Base);

BEGIN {
    my $path = Module::Metadata->find_module_by_name(__PACKAGE__);
    $path =~ s!\.pm$!/lib!;
    unshift @INC, $path
        unless grep { $_ eq $path } @INC;

    # Only DBIC schema registration belongs in BEGIN
    require Koha::Schema::Result::MyPluginRecord;
    Koha::Schema->register_class(
        MyPluginRecord => 'Koha::Schema::Result::MyPluginRecord' );
    Koha::Database->schema( { new => 1 } );
}

# Factory methods — lazy-load via require
sub records {
    require MyPlugin::Records;
    return MyPlugin::Records->new;
}

sub find_record {
    my ( $self, $id ) = @_;
    require MyPlugin::Records;
    return MyPlugin::Records->find($id);
}

sub new_record {
    my ( $self, $data ) = @_;
    require MyPlugin::Record;
    return MyPlugin::Record->new($data);
}

sub new_client {
    my ( $self, %args ) = @_;
    require MyPlugin::Client;
    return MyPlugin::Client->new(%args);
}
```

**Controller — uses only the plugin class:**
```perl
package Koha::Plugin::Com::Company::MyPlugin::Controller;

use Mojo::Base 'Mojolicious::Controller';
use Koha::Plugin::Com::Company::MyPlugin;

my $plugin = Koha::Plugin::Com::Company::MyPlugin->new;

sub list {
    my $c = shift->openapi->valid_input or return;
    return $c->render(
        status  => 200,
        openapi => $c->objects->search( $plugin->records ),
    );
}

sub get {
    my $c = shift->openapi->valid_input or return;
    my $record = $plugin->find_record( $c->param('record_id') );
    # ...
}

sub add {
    my $c = shift->openapi->valid_input or return;
    my $record = $plugin->new_record( $c->req->json )->store;
    # ...
}
```

### Key rules

1. **BEGIN block**: only `@INC` setup and DBIC schema registration — nothing else
2. **Guard `@INC`**: check for duplicates before `unshift` to handle symlinked plugin dirs
3. **Factory methods**: use `require` (not `use`) inside the sub body — Perl's `require` is a no-op on subsequent calls thanks to `%INC`
4. **Controller**: `use` only the plugin class, never the lib classes directly
5. **Plugin instance**: instantiate once at module scope (`my $plugin = ...->new`) and reuse across controller methods

This pattern eliminates double-loading entirely because each library class is only ever `require`d through one code path — the factory method.

## Best practices

### Configuration management

1. **Always validate configuration** before storing
2. **Use caching** for processed configuration
3. **Provide defaults** for all optional settings
4. **Clear cache** when configuration changes
5. **Support configuration export/import** for deployment

### Data storage

1. **Use meaningful keys** for stored data
2. **Version your data structures** for upgrades
3. **Clean up data** in uninstall method
4. **Use transactions** for complex operations
5. **Handle serialization errors** gracefully

### Error handling

1. **Validate all inputs** before processing
2. **Provide meaningful error messages** to users
3. **Log errors** for debugging
4. **Graceful degradation** when possible
5. **Rollback on failure** for critical operations

### Performance

1. **Cache expensive operations** using store_data/retrieve_data
2. **Use database indexes** for plugin tables
3. **Minimize database queries** in loops
4. **Clean up temporary data** regularly
5. **Profile plugin performance** impact

## Version management

**Version Numbering:**
- Follow semantic versioning (MAJOR.MINOR.PATCH)
- Use `npm version patch|minor|major` to bump version, update the plugin `.pm` file, and create a git tag
- Consolidate related changes in the same version

**Changelog Management:**

Use [Keep a Changelog](https://keepachangelog.com/) format. Every released version must have an entry — never skip versions. Reference issue numbers in each entry.

```markdown
## [1.0.2] - 2026-03-31

### Added
- File bundle step in publish wizard (#21)

### Fixed
- Duplicate metadata in DSpace deposits (#19)
```

**Plugin Commit Format:**
```
[#issue_number] Description of change
```

Multiple issues in one commit are fine: `[#17] Foo [#19] Bar`

## CI/CD & automation

### GitHub Actions workflow

**Multi-Version Testing Matrix:**
```yaml
strategy:
  matrix:
    koha-version: [main, stable, oldstable]

steps:
  - name: Launch KTD
    run: |
      cd ../koha-testing-docker
      ktd --name ci --plugins up -d
      ktd --name ci --wait-ready 120

  - name: Install Plugin
    run: |
      ktd --name ci --shell --run "cd /kohadevbox/koha && perl misc/devel/install_plugins.pl"

  - name: Run Tests
    run: |
      ktd --name ci --shell --run "
        cd /kohadevbox/plugins/plugin-name &&
        export PERL5LIB=\$PERL5LIB:. &&
        prove -v t/
      "
```

**Key CI Configurations:**
- Test on every push (not just main branch)
- Package only on version tags (`v*.*.*`)
- Use proper KTD environment
- Include comprehensive error logging on failure

### Packaging

**KPZ Build with Gulp:**
```javascript
async function build() {
  await execPromise("mkdir -p dist");
  await execPromise("cp -r Koha dist/.");
  await execPromise(`sed -i -e "s/1970-01-01/${today}/g" ${pm_file_path_dist}`);
  await execPromise(`cd dist && zip -r ../${release_filename} ./Koha`);
  await execPromise("rm -rf dist");
}
```

**Packaging Rules:**
- Only include `Koha/` directory in the KPZ
- Exclude development files (tests, docs, configs)
- `.gitignore` and packaging are separate systems

**Release Workflow:**
```bash
npm version patch   # bumps version, updates .pm, creates git tag
git push --follow-tags  # triggers CI release job
```

This architecture provides a robust foundation for building maintainable, scalable Koha plugins with proper configuration management, data persistence, and integration with the core Koha system.
