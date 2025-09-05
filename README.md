# Koha Development Handbook

A comprehensive guide for Koha and plugin development, covering architecture, design patterns, coding standards, testing methodologies, and development environment setup.

## Table of Contents

1. [Development Environment](#development-environment)
2. [Architecture & Design Patterns](#architecture--design-patterns)
3. [Coding Standards](#coding-standards)
4. [Testing Framework](#testing-framework)
5. [Plugin Development](#plugin-development)
6. [Commit Standards](#commit-standards)
7. [CI/CD & Automation](#cicd--automation)
8. [Operational Deployment](#operational-deployment)

## Development Environment

### Koha Testing Docker (KTD) Setup

**Essential Environment Variables:**
```bash
export KTD_HOME=/path/to/koha-testing-docker
export PLUGINS_DIR=/path/to/plugins/parent/directory
export SYNC_REPO=/path/to/koha/source
export LOCAL_USER_ID=$(id -u)
```

**KTD Workflow:**
```bash
# 1. Launch with plugins support
ktd --name instance --plugins up -d

# 2. Wait for readiness
ktd --name instance --wait-ready 120

# 3. Install plugins
ktd --name instance --shell --run "cd /kohadevbox/koha && perl misc/devel/install_plugins.pl"

# 4. Run commands in container
ktd --name instance --shell --run "command"
```

**Critical Setup Requirements:**
- `.env` file must exist (copy from `env/defaults.env`)
- `ktd` script location: `$KTD_HOME/bin/ktd`
- All commands require `KTD_HOME` environment variable
- Plugin directory structure: `/kohadevbox/plugins/plugin-name/`

### Development Container Environment

**Standard PERL5LIB Setup:**
```bash
export PERL5LIB=/kohadevbox/koha:/kohadevbox/plugins/plugin-name/Koha/Plugin/Com/Company/PluginName/lib:/kohadevbox/plugins/plugin-name:.
```

**Common Development Commands:**
```bash
# Format code with Koha standards
/kohadevbox/koha/misc/devel/tidy.pl path/to/file.pm

# Run tests
prove -v t/ t/db_dependent/

# Install plugins
cd /kohadevbox/koha && perl misc/devel/install_plugins.pl
```

## Architecture & Design Patterns

### ActionHandler Pattern (Rapido ILL Example)

**Separation of Concerns:**
- **Backend**: UI wrapper, handles user interface flow and return structures
- **ActionHandlers**: Business logic, database operations, API calls
- **Client**: External API communication layer

**Implementation Pattern:**
```perl
# Backend delegates to ActionHandler
sub backend_method {
    my ( $self, $params ) = @_;
    
    my $request = $params->{request};
    my $pod     = $self->{plugin}->get_req_pod($request);
    
    return try {
        $self->{plugin}->get_action_handler($pod)->business_method($request);
        
        return {
            error   => 0,
            method  => 'backend_method',
            stage   => 'commit',
            next    => 'illview',
        };
    } catch {
        return {
            error   => 1,
            message => "$_",
            method  => 'backend_method',
        };
    };
}
```

**ActionHandler Business Logic:**
```perl
sub business_method {
    my ( $self, $req, $params ) = @_;
    
    $params //= {};
    my $options = $params->{client_options} // {};
    
    return try {
        Koha::Database->schema->storage->txn_do(
            sub {
                # Business logic here
                $self->{plugin}->get_client($self->{pod})->api_call($data, $options);
                $req->status('NEW_STATUS')->store;
            }
        );
        return $self;
    } catch {
        RapidoILL::Exception->throw(
            sprintf("Unhandled exception: %s", $_)
        );
    };
}
```

### Database Schema Patterns

**Koha Object Model:**
```perl
# Single object class
package Koha::Plugin::MyPlugin::MyObject;
use base qw(Koha::Object);
sub _type { return 'PluginMyobject'; }

# Collection class  
package Koha::Plugin::MyPlugin::MyObjects;
use base qw(Koha::Objects);
sub _type { return 'Koha::Plugin::MyPlugin::MyObject'; }
```

**Schema Registration:**
```perl
# In plugin's BEGIN block
BEGIN {
    push @INC, dirname(__FILE__) . '/lib';
}

# Schema classes auto-generated and registered
use Koha::Schema;
Koha::Schema->register_class('PluginMyobject', 'Koha::Schema::Result::PluginMyobject');
```

### Koha::Object System

**Core Concepts:**
- **Koha::Object**: Base class for single database record objects
- **Koha::Objects**: Base class for collections (result sets) of objects
- **DBIx::Class Integration**: Built on top of DBIx::Class ORM
- **Automatic Methods**: Provides CRUD operations and relationship handling

**Object Class Pattern:**
```perl
package Koha::MyRecord;

use Modern::Perl;
use base qw(Koha::Object);

=head1 NAME

Koha::MyRecord - Koha object class for records

=head1 API

=head2 Class methods

=head3 _type

Return the DBIx::Class result name for this object

=cut

sub _type {
    return 'Myrecord';  # Must match schema class name
}

=head2 Instance methods

=head3 custom_method

Custom business logic method

=cut

sub custom_method {
    my ($self) = @_;
    
    # Access fields directly
    my $id = $self->id;
    my $name = $self->name;
    
    # Modify and save
    $self->status('PROCESSED')->store;
    
    return $self;
}

1;
```

**Collection Class Pattern:**
```perl
package Koha::MyRecords;

use Modern::Perl;
use base qw(Koha::Objects);

=head1 NAME

Koha::MyRecords - Koha objects class for records

=head1 API

=head2 Class methods

=head3 _type

Return the object class name

=cut

sub _type {
    return 'Koha::MyRecord';
}

=head2 Instance methods

=head3 pending

Return only pending records

=cut

sub pending {
    my ($self) = @_;
    
    return $self->search({ status => 'PENDING' });
}

1;
```

**Schema Class (Auto-generated):**
```perl
package Koha::Schema::Result::Myrecord;

use strict;
use warnings;
use base 'DBIx::Class::Core';

__PACKAGE__->table('myrecords');

__PACKAGE__->add_columns(
    'id' => {
        data_type         => 'integer',
        is_auto_increment => 1,
        is_nullable       => 0,
    },
    'name' => {
        data_type   => 'varchar',
        size        => 255,
        is_nullable => 0,
    },
    'status' => {
        data_type     => 'varchar',
        size          => 50,
        is_nullable   => 0,
        default_value => 'PENDING',
    },
    'created_on' => {
        data_type     => 'timestamp',
        is_nullable   => 0,
        default_value => \'CURRENT_TIMESTAMP',
    },
);

__PACKAGE__->set_primary_key('id');

1;
```

**Usage Patterns:**

**Creating Objects:**
```perl
# Create new record
my $record = Koha::MyRecord->new({
    name   => 'Test Record',
    status => 'PENDING'
})->store();

# Alternative via collection
my $records = Koha::MyRecords->new();
my $record = $records->_resultset->create({
    name   => 'Test Record',
    status => 'PENDING'
});
```

**Finding Objects:**
```perl
# Find by primary key
my $record = Koha::MyRecord->find($id);

# Search with conditions
my $records = Koha::MyRecords->search({
    status => 'PENDING'
});

# Chain methods
my $pending_count = Koha::MyRecords->new()->pending()->count();
```

**Updating Objects:**
```perl
# Update single field
$record->status('PROCESSED')->store();

# Update multiple fields
$record->set({
    status      => 'COMPLETED',
    processed_on => dt_from_string()
})->store();

# Bulk update via collection
Koha::MyRecords->search({ status => 'PENDING' })
               ->update({ status => 'CANCELLED' });
```

**Relationships:**
```perl
# Define relationships in schema class
__PACKAGE__->belongs_to(
    'patron',
    'Koha::Schema::Result::Borrower',
    { 'foreign.borrowernumber' => 'self.borrowernumber' }
);

# Use relationships in object class
sub patron {
    my ($self) = @_;
    return Koha::Patron->_new_from_dbic($self->_result->patron);
}

# Access related objects
my $patron = $record->patron();
my $library = $patron->library();
```

**Best Practices:**

1. **Naming Conventions**:
   - Object class: Singular noun (`Koha::MyRecord`)
   - Collection class: Plural noun (`Koha::MyRecords`)
   - Schema class: Lowercase (`Koha::Schema::Result::Myrecord`)

2. **Method Organization**:
   - **Class methods**: Return types, validation, factory methods
   - **Instance methods**: Business logic, custom operations
   - **Collection methods**: Filtering, bulk operations

3. **Error Handling**:
   ```perl
   # Check if object exists
   my $record = Koha::MyRecord->find($id);
   return unless $record;
   
   # Handle database errors
   try {
       $record->store();
   } catch {
       warn "Failed to save record: $_";
       return;
   };
   ```

4. **Performance Considerations**:
   ```perl
   # Use prefetch for related data
   my $records = Koha::MyRecords->search({}, { prefetch => 'patron' });
   
   # Use result set methods for bulk operations
   $records->delete();  # More efficient than iterating
   ```

**Schema Generation Workflow:**
1. Create/modify database tables
2. Regenerate schema using KTD: `misc/devel/update_dbix_class_files.pl`
3. Copy generated schema files to appropriate locations
4. Create corresponding Koha::Object and Koha::Objects classes

### Plugin-Specific Koha::Object Implementation

**Plugin Schema Considerations:**
- Plugin tables require custom schema file generation
- Plugin objects follow same patterns but with plugin-specific naming
- Schema registration happens in plugin's BEGIN block
- Plugin objects are not part of core Koha schema

**Plugin Object Naming Pattern:**
```perl
# Plugin object class
package Koha::Plugin::Com::Company::PluginName::MyRecord;
use base qw(Koha::Object);
sub _type { return 'PluginMyrecord'; }

# Plugin collection class
package Koha::Plugin::Com::Company::PluginName::MyRecords;
use base qw(Koha::Objects);
sub _type { return 'Koha::Plugin::Com::Company::PluginName::MyRecord'; }
```

**Plugin Schema Registration:**
```perl
# In plugin's main class BEGIN block
BEGIN {
    push @INC, dirname(__FILE__) . '/lib';
}

# Register plugin schema classes
use Koha::Schema;
Koha::Schema->register_class('PluginMyrecord', 'Koha::Schema::Result::PluginMyrecord');
```

**Plugin Schema File Generation:**
*Note: Detailed workflow for generating plugin schema files will be documented here.*

### Configuration Management

**YAML-based Plugin Configuration:**
```perl
sub configuration {
    my ($self) = @_;
    
    my $config_yaml = $self->retrieve_data('configuration') || '';
    return {} unless $config_yaml;
    
    my $config = try {
        YAML::XS::Load($config_yaml);
    } catch {
        warn "Invalid YAML configuration: $_";
        return {};
    };
    
    # Apply defaults and transformations
    foreach my $pod_name (keys %$config) {
        my $pod_config = $config->{$pod_name};
        
        # Set defaults
        $pod_config->{debt_blocks_holds} //= 1;
        $pod_config->{max_debt_blocks_holds} //= 100;
        
        # Transform library mappings
        if ($pod_config->{library_to_location}) {
            # Transform mapping structure
        }
    }
    
    return $config;
}
```

## Coding Standards

### Code Formatting

**Mandatory**: Use Koha's tidy.pl for all Perl code:
```bash
# Format single file
/kohadevbox/koha/misc/devel/tidy.pl path/to/file.pm

# Format all plugin files
find Koha/Plugin/ -name "*.pm" -exec /kohadevbox/koha/misc/devel/tidy.pl {} \;

# Always remove backup files
find . -name "*.bak" -delete
```

**Pre-Commit Workflow:**
1. Make code changes
2. Format with tidy.pl
3. Remove .bak files
4. Run tests to verify
5. Commit clean code

### Perl Best Practices

**Modern Perl Usage:**
```perl
use Modern::Perl;
use Try::Tiny qw(catch try);  # Always use Try::Tiny, never eval
use Koha::Database;
use Koha::DateUtils qw( dt_from_string );
```

**Exception Handling:**
```perl
# ✅ CORRECT - Use Try::Tiny
return try {
    # Code that might fail
} catch {
    # Handle exception
    warn "Error: $_";
    return { error => 1, message => "$_" };
};

# ❌ WRONG - Don't use eval
eval {
    # Code
};
if ($@) {
    # Handle error
}
```

**Database Transactions:**
```perl
# Always wrap database operations in transactions
Koha::Database->schema->storage->txn_do(
    sub {
        # Multiple database operations
        $object->store;
        $related->update;
    }
);
```

### Test Naming Conventions

**Subtest Titles:**
- Format: `'method_name() tests'`
- Always include parentheses and use "tests" (plural)
- Examples:
  ```perl
  subtest 'item_received() tests' => sub { ... };
  subtest 'renewal_request() tests' => sub { ... };
  ```

**Test File Organization:**
- Unit tests: `t/`
- Database-dependent tests: `t/db_dependent/`
- Class-based naming: `t/db_dependent/ClassName.t`

## Testing Framework

### Test Structure Patterns

**File Organization Standards:**
- Database-dependent tests for `a_method` in class `Some::Class` → `t/db_dependent/Some/Class.t`
- Main subtest titled `'a_method() tests'` contains all tests for that method
- Inner subtests have descriptive titles for specific behaviors

**Standard Test File Structure:**
```perl
use Modern::Perl;
use Test::More tests => N;  # N = number of main subtests + use_ok
use Test::Exception;
use Test::MockModule;
use Test::MockObject;

use t::lib::TestBuilder;
use t::lib::Mocks;
use t::lib::Mocks::Logger;

BEGIN {
    use_ok('Some::Class');
}

# Global variables for entire test file
my $schema  = Koha::Database->new->schema;
my $builder = t::lib::TestBuilder->new;
my $logger  = t::lib::Mocks::Logger->new();

subtest 'a_method() tests' => sub {
    plan tests => 3;  # Number of individual tests
    
    $schema->storage->txn_begin;
    $logger->clear();
    
    # Test implementation - all tests for this method
    
    $schema->storage->txn_rollback;
};

# OR if multiple behaviors need testing:

subtest 'a_method() tests' => sub {
    plan tests => 2;  # Number of inner subtests
    
    subtest 'Successful operations' => sub {
        plan tests => 3;  # Number of individual tests
        
        $schema->storage->txn_begin;
        $logger->clear();
        
        # Test implementation
        
        $schema->storage->txn_rollback;
    };
    
    subtest 'Error conditions' => sub {
        plan tests => 2;
        
        $schema->storage->txn_begin;
        
        # Error test implementation
        
        $schema->storage->txn_rollback;
    };
};
```

**Transaction Rules:**
- Main subtest wrapped in transaction if only one behavior tested
- Each inner subtest wrapped in transaction if multiple behaviors tested
- Never nest transactions

**TestBuilder Class Naming Convention:**
```perl
# IMPORTANT: TestBuilder->build_object() requires PLURAL class names
my $patron = $builder->build_object({ class => 'Koha::Patrons' });     # ✓ Correct
my $library = $builder->build_object({ class => 'Koha::Libraries' });   # ✓ Correct  
my $item = $builder->build_object({ class => 'Koha::Items' });         # ✓ Correct

# NOT singular class names
my $patron = $builder->build_object({ class => 'Koha::Patron' });      # ✗ Wrong
```

**Rule**: Always use the plural/collection class name (e.g., `Koha::Patrons`, `RapidoILL::CircActions`) not the singular object class name (e.g., `Koha::Patron`, `RapidoILL::CircAction`).

**Database-Dependent Test Template:**
```perl
use Modern::Perl;
use Test::More tests => 2;  # use_ok + main subtest
use Test::Exception;
use Test::MockModule;
use Test::MockObject;
use Try::Tiny qw(catch try);

use t::lib::TestBuilder;
use t::lib::Mocks;
use t::lib::Mocks::Logger;  # Logger mock for testing log output

BEGIN {
    use_ok('Module::Under::Test');
}

my $schema = Koha::Database->new->schema;
my $builder = t::lib::TestBuilder->new;

subtest 'method_name() tests' => sub {
    plan tests => 3;  # Number of subtests
    
    subtest 'Successful operations' => sub {
        plan tests => 4;  # Number of individual tests
        
        $schema->storage->txn_begin;
        
        # Test setup
        my $object = $builder->build_object({ class => 'Koha::Objects' });
        
        # Test execution
        my $result;
        lives_ok {
            $result = $object->method_under_test();
        } 'Method executes without error';
        
        # Assertions
        is($result->status, 'EXPECTED', 'Status set correctly');
        
        $schema->storage->txn_rollback;
    };
    
    subtest 'Error conditions' => sub {
        plan tests => 2;
        
        $schema->storage->txn_begin;
        
        throws_ok {
            # Code that should throw exception
        } qr/Expected error/, 'Throws expected exception';
        
        $schema->storage->txn_rollback;
    };
    
    subtest 'Edge cases' => sub {
        # Additional test scenarios
    };
};
```

### Exception Testing Patterns

**Exception::Class Testing:**
```perl
# Test exception throwing
throws_ok {
    MyException->throw(field => 'value');
} 'MyException', 'Exception can be thrown';

# Test exception properties
my $exception;
try {
    MyException->throw(field => 'value');
} catch {
    $exception = $_;
};

isa_ok($exception, 'MyException', 'Exception has correct class');
is($exception->field, 'value', 'Exception field set correctly');
```

### Mocking Patterns

**Mock External Dependencies:**
```perl
# Mock plugin methods
my $plugin_module = Test::MockModule->new('Plugin::Class');
$plugin_module->mock('external_method', sub { return 'mocked_result'; });

# Mock objects
my $mock_client = Test::MockObject->new();
$mock_client->mock('api_call', sub { 
    my ($self, $data) = @_;
    # Track calls or return test data
    return { success => 1 };
});
```

### Test Configuration

**Disable External Calls:**
```perl
# Set dev_mode in test configurations
my $config = {
    'test-pod' => {
        dev_mode => 1,  # Disables external API calls
        base_url => 'https://test.example.com',
        # ... other config
    }
};
```

### Logger Testing with t::lib::Mocks::Logger

**Setup and Basic Usage:**
```perl
use t::lib::Mocks::Logger;

# Create logger mock instance (usually at test file level)
my $logger = t::lib::Mocks::Logger->new();

# The mock automatically replaces Koha::Logger->get()
my $mocked_logger = Koha::Logger->get();

# Clear previous log messages before test
$logger->clear();

# Your code that logs messages
$mocked_logger->debug('Debug message');
$mocked_logger->info('Processing item 123');
$mocked_logger->warn('Item not found');
$mocked_logger->error('Database connection failed');
```

**Testing Log Messages:**

**Exact Message Matching:**
```perl
# Test exact log message content
$logger->debug_is('Debug message', 'Debug message logged correctly');
$logger->info_is('Processing item 123', 'Info message matches');
$logger->warn_is('Item not found', 'Warning message captured');
$logger->error_is('Database connection failed', 'Error logged');
```

**Regex Pattern Matching:**
```perl
# Test log messages with patterns
$logger->debug_like(qr/Debug/, 'Debug message contains expected text');
$logger->info_like(qr/Processing item \d+/, 'Info message matches pattern');
$logger->warn_like(qr/Item.*not found/, 'Warning matches regex');
$logger->error_like(qr/Database.*failed/, 'Error message pattern matched');
```

**Message Counting and Consumption:**
```perl
# Count total messages
is($logger->count, 4, 'Four messages logged total');

# Count by log level
is($logger->count('debug'), 1, 'One debug message');
is($logger->count('info'), 1, 'One info message');

# Messages are consumed when tested (FIFO queue)
$mocked_logger->debug('First debug');
$mocked_logger->debug('Second debug');

$logger->debug_is('First debug', 'First message consumed');
$logger->debug_is('Second debug', 'Second message consumed');
is($logger->count('debug'), 0, 'All debug messages consumed');
```

**Method Chaining and Debugging:**
```perl
# Chain multiple assertions
$logger->debug_is('Debug message', 'Debug test')
       ->info_is('Info message', 'Info test')
       ->warn_like(qr/Warning/, 'Warning pattern test');

# Clear messages (all or by level)
$logger->clear();           # Clear all messages
$logger->clear('debug');    # Only clear debug messages

# Debug test failures
$logger->diag();  # Prints all captured messages to test output
```

**Complete Logger Test Example:**
```perl
subtest 'Logger testing example' => sub {
    plan tests => 4;
    
    $logger->clear();  # Always clear before test
    
    # Code under test that logs messages
    my $processor = MyModule->new();
    $processor->process_item(123);
    
    # Test expected messages
    $logger->info_like(qr/Processing item 123/, 'Processing logged');
    $logger->debug_is('Item validation passed', 'Validation logged');
    is($logger->count, 2, 'Two messages logged total');
    
    # Test error path
    $processor->process_item('invalid');
    $logger->error_like(qr/Invalid item/, 'Error logged for invalid input');
};
```

## Plugin Development

### Plugin Structure

**Standard Plugin Directory Layout:**
```
Koha/Plugin/Com/Company/PluginName.pm          # Main plugin class
Koha/Plugin/Com/Company/PluginName/lib/        # Plugin libraries
├── PluginName/                                # Business logic
│   ├── Backend.pm                            # ILL Backend (if applicable)
│   ├── Client.pm                             # API client
│   ├── Exceptions.pm                         # Custom exceptions
│   └── Backend/                              # ActionHandlers
│       ├── LenderActions.pm                  # Lender business logic
│       └── BorrowerActions.pm                # Borrower business logic
├── Koha/Schema/Result/                       # Database schema classes
└── templates/                                # Template files
t/                                            # Unit tests
t/db_dependent/                               # Integration tests
scripts/                                      # System scripts
```

### Plugin Lifecycle Methods

**Essential Plugin Methods:**
```perl
package Koha::Plugin::Com::Company::PluginName;
use base qw(Koha::Plugins::Base);

sub new {
    my ($class, $args) = @_;
    $args->{'metadata'} = {
        name            => 'Plugin Name',
        version         => '1.0.0',
        minimum_version => '22.11.00.000',
    };
    return $class->SUPER::new($args);
}

sub install {
    my ($self) = @_;
    # Create database tables, set defaults
}

sub upgrade {
    my ($self, $args) = @_;
    my $dt = dt_from_string();
    $self->store_data({ last_upgraded => $dt->ymd('-') . ' ' . $dt->hms(':') });
}

sub uninstall {
    my ($self) = @_;
    # Cleanup: drop tables, remove data
}
```

### Configuration Management

**YAML Configuration Pattern:**
```perl
sub configuration {
    my ($self) = @_;
    
    my $cached_config = $self->retrieve_data('cached_configuration');
    return $cached_config if $cached_config;
    
    my $config_yaml = $self->retrieve_data('configuration') || '';
    my $config = YAML::XS::Load($config_yaml) || {};
    
    # Apply defaults and transformations
    foreach my $pod_name (keys %$config) {
        $self->_apply_config_defaults($config->{$pod_name});
        $self->_transform_config($config->{$pod_name});
    }
    
    $self->store_data({ cached_configuration => $config });
    return $config;
}
```

## Commit Standards

### Rapido ILL Commit Format

**Subject Line Format:**
```
[#issue_number] Description of change
```

**Examples:**
```
[#65] Implement ActionHandler system for task_queue_daemon.pl integration
[#42] Fix authentication token refresh in APIHttpClient  
[#73] Add comprehensive exception handling for CircAction processing
```

**Complete Commit Structure:**
```
[#65] Implement ActionHandler system for task_queue_daemon.pl integration

Detailed explanation of the enhancement/fix.

Changes:
* Bullet point list of specific changes
* Each change should be clear and specific

Benefits:
* List the benefits achieved
* Focus on architectural improvements

To test:
1. Step-by-step instructions
2. Run: ktd --name rapido --shell
   k$ command_to_run
=> SUCCESS: Expected positive outcome
3. Sign off :-D
```

### Version Management

**Version Numbering Rules:**
- **Never change versions** without explicit instruction
- **Add to current version** until told to bump
- **Follow semantic versioning** when bumping (MAJOR.MINOR.PATCH)
- **Consolidate related changes** in same version

**Changelog Management:**
```markdown
## [0.9.4] - 2025-09-03

### Added
- [#99] New feature description

### Changed  
- [#98] Modification description
- [#90] Refactoring description

### Fixed
- [#93] Bug fix description

### Removed
- [#92] Deprecated feature removal
```

## CI/CD & Automation

### GitHub Actions Workflow

**Multi-Version Testing Matrix:**
```yaml
strategy:
  matrix:
    koha-version: [main, stable, oldstable]

steps:
  - name: Setup Environment
    run: |
      echo "PLUGINS_DIR=$(pwd)/.." >> $GITHUB_ENV
      echo "SYNC_REPO=$(pwd)/../kohaclone" >> $GITHUB_ENV
      echo "KTD_HOME=$(pwd)/../koha-testing-docker" >> $GITHUB_ENV

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
      ktd --name ci --shell --run "cd /kohadevbox/plugins/plugin-name && export PERL5LIB=/kohadevbox/koha:/kohadevbox/plugins/plugin-name/lib:. && prove -lr t/"
```

**Key CI Configurations:**
- Test on every push (not just main branch)
- Package only on version tags (`v*.*.*`)
- Use proper KTD environment without sudo
- Include comprehensive error logging

### Packaging

**Automated Packaging with Gulp:**
```javascript
// gulpfile.js pattern
gulp.task('build', function() {
    return gulp.src(['Koha/**/*'])
        .pipe(zip('plugin-name.kpz'))
        .pipe(gulp.dest('dist/'));
});
```

**Packaging Rules:**
- Only include `Koha/` directory in package
- Exclude development files (tests, docs, configs)
- `.gitignore` and packaging are separate systems
- Create `.kpz` files for distribution

## Operational Deployment

### System Services

**Task Queue Daemon (systemd):**
```ini
[Unit]
Description=Plugin Task Queue Daemon
After=network.target

[Service]
Type=simple
User=koha-instance
Environment=KOHA_INSTANCE=instance_name
ExecStart=/usr/bin/perl /path/to/plugin/scripts/task_queue_daemon.pl
Restart=always

[Install]
WantedBy=multi-user.target
```

### Cron Jobs

**Synchronization Scripts:**
```bash
# One entry per pod, every 5 minutes
*/5 * * * * cd /var/lib/koha/<instance>/plugins; PERL5LIB=/usr/share/koha/lib:Koha/Plugin/Com/Company/PluginName/lib:. perl Koha/Plugin/Com/Company/PluginName/scripts/sync_requests.pl --pod <pod_name>
```

**Variables:**
- `<instance>`: Koha instance name (e.g., `kohadev`, `library`)
- `<pod_name>`: Pod identifier from configuration

### Monitoring & Debugging

**Log Analysis:**
```bash
# Plugin-specific logs
tail -f /var/log/koha/<instance>/plack-intranet-error.log | grep PluginName

# Task queue logs  
journalctl -u plugin-task-queue -f

# Cron job logs
grep "sync_requests" /var/log/syslog
```

**Configuration Validation:**
```perl
# Test configuration loading
my $plugin = Koha::Plugin::Com::Company::PluginName->new();
my $config = $plugin->configuration();
use Data::Dumper;
print Dumper($config);
```

## Best Practices Summary

### Development Workflow
1. **Setup KTD** with proper environment variables
2. **Format code** with tidy.pl before every commit
3. **Write tests first** (TDD approach when possible)
4. **Use transactions** for all database operations
5. **Mock external dependencies** in tests
6. **Follow naming conventions** for tests and methods

### Architecture Principles
1. **Separation of concerns** (Backend ↔ ActionHandlers ↔ Client)
2. **Consistent error handling** with Try::Tiny and custom exceptions
3. **Configuration-driven behavior** with YAML and defaults
4. **Transaction safety** for all database operations
5. **Comprehensive logging** for debugging and monitoring

### Code Quality
1. **Mandatory code formatting** with Koha standards
2. **Comprehensive test coverage** (unit + integration)
3. **Proper exception handling** throughout codebase
4. **Clear documentation** and inline comments
5. **Consistent commit messages** with issue tracking

This handbook serves as the definitive guide for Koha and plugin development, ensuring consistency, quality, and maintainability across all projects.
