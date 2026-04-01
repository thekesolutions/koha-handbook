# Koha development handbook

A comprehensive guide for Koha and plugin development, covering architecture, design patterns, coding standards, testing methodologies, and development environment setup.

## Table of contents

1. [Development environment](#development-environment)
2. [Architecture & design patterns](#architecture--design-patterns)
3. [Background jobs system](background_jobs.md)
4. [Coding standards](#coding-standards)
5. [Testing framework](#testing-framework)
6. [Plugin development](#plugin-development)
7. [Commit standards](#commit-standards)
8. [Operational deployment](#operational-deployment)

## Development environment

### Koha Testing Docker (KTD) Setup

**Essential Environment Variables:**
```bash
export KTD_HOME=/path/to/koha-testing-docker
export SYNC_REPO=/path/to/koha/source
export LOCAL_USER_ID=$(id -u)
```

**KTD Workflow:**
```bash
# 1. Start the proxy (once, shared across instances)
ktd_proxy --start

# 2. Launch
ktd --proxy up -d

# 2. Wait for readiness
ktd --wait-ready 120

# 3. Run commands in container
ktd --shell --run "command"
```

**KTD Database Management:**
```bash
# Run database updates (atomic updates)
ktd --shell --run "cd /kohadevbox/koha && perl installer/data/mysql/updatedatabase.pl"

# Access MySQL shell for Koha database
ktd --shell --run "koha-mysql kohadev"

# Execute SQL commands directly
ktd --shell --run "koha-mysql kohadev -e 'SHOW TRIGGERS LIKE \"borrowers\";'"

# Database backup and restore testing
ktd --shell --run "mysqldump -hdb -uroot -ppassword koha_kohadev > /tmp/backup.sql"
ktd --shell --run "mysql -hdb -uroot -ppassword -e 'CREATE DATABASE test_restore;'"
ktd --shell --run "mysql -hdb -uroot -ppassword test_restore < /tmp/backup.sql"
ktd --shell --run "mysql -hdb -uroot -ppassword -e 'DROP DATABASE test_restore;'"
```

**Critical Setup Requirements:**
- `.env` file must exist (copy from `env/defaults.env`)
- `ktd` script location: `$KTD_HOME/bin/ktd`
- All commands require `KTD_HOME` environment variable

### Common development commands

```bash
# Format code with Koha standards
/kohadevbox/koha/misc/devel/tidy.pl path/to/file.pm

# Run tests
prove -v t/ t/db_dependent/
```

## Architecture & design patterns

### Koha objects and DBIx::Class integration

For comprehensive understanding of Koha's object-relational mapping system, see:
**[Koha Objects System: DBIx::Class Integration Architecture](koha_objects_system.md)**

This guide covers:
- DBIx::Class Schema layer and Koha Object wrapper relationships
- Schema regeneration tools (`update_dbic_class_files.pl`, KTD `dbic`)
- Object creation patterns, database operations, and performance considerations
- Plugin development with custom objects and schema extensions

### Template::Toolkit system architecture

For detailed understanding of Koha's templating and internationalization system, see:
**[Koha Template::Toolkit System Architecture](koha_template_toolkit.md)**

This guide covers:
- C4::Templates and C4::Languages integration for multi-theme, multi-language support
- Template resolution process, fallback mechanisms, and directory structure
- Internationalization (i18n) patterns, translation workflows, and language detection
- Plugin template integration, performance optimization, and development best practices

### RESTful API architecture

For comprehensive understanding of Koha's REST API built with Mojolicious and OpenAPI, see:
**[Koha RESTful API Architecture: Mojolicious and OpenAPI Integration](koha_rest_api_architecture.md)**

This guide covers:
- Mojolicious framework integration with OpenAPI specification validation
- Custom helper plugins (Objects, Query, Pagination, Exceptions) and Koha Object system integration
- Authentication/authorization patterns, plugin route registration, and API extension mechanisms
- Performance optimizations, development patterns, and advanced features (streaming, bulk operations)

### Search architecture

For detailed understanding of Koha's Elasticsearch integration and search system, see:
**[Koha Search Architecture: Elasticsearch Integration and Field Mapping](koha_search_architecture.md)**

This guide covers:
- ElasticsearchMARCFormat options (base64ISO2709 vs ARRAY) and their implications
- Search field mapping system, database structure, and query behavior
- Standard vs whole record search functionality and 856 field searchability
- Performance considerations, troubleshooting, and best practices for search optimization

### Koha::Object System

For comprehensive understanding of Koha's object-relational mapping, see:
**[Koha Objects System: DBIx::Class Integration Architecture](koha_objects_system.md)**

For plugin-specific Koha::Object patterns (schema registration, naming conventions, factory methods), see:
**[Koha Plugin Architecture](plugin_architecture.md)**

### Configuration management

See **[Koha Plugin Architecture — Configuration Management](plugin_architecture.md)** for YAML-based configuration patterns, caching, validation, and management scripts.

## Coding standards

### GPL license headers

**All Koha files must include the standard GPL header:**
```perl
#!/usr/bin/perl

# This file is part of Koha.
#
# Koha is free software; you can redistribute it and/or modify it
# under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 3 of the License, or
# (at your option) any later version.
#
# Koha is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with Koha; if not, see <https://www.gnu.org/licenses>.
```

**Requirements:**
- Must use HTTPS URL: `<https://www.gnu.org/licenses>`
- Required for all `.pm`, `.pl`, and `.t` files
- Place immediately after shebang line

### Code formatting

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

### TODO item management

**Tracking TODO Items in Commits:**
When commits include TODO sections, track progress systematically:

```
TODO:
* Exception handling is not implemented in Koha::Patron->store() ✅ COMPLETED
* Exceptions are not defined yet ✅ COMPLETED  
* Method validation needs updating ✅ COMPLETED
* Some existing tests should fail and will need tweaks ⏳ IN PROGRESS
* Database consistency check needed ⏳ PENDING
```

**Follow-up Commit Pattern for TODO Resolution:**
```
Bug XXXXX: (follow-up) Complete TODO items for [feature]

This patch completes the remaining TODO items from the original implementation.

Changes:
- Add exception classes for proper error handling
- Update validation methods to work with database constraints  
- Adapt existing tests to new behavior patterns
- Add comprehensive test coverage

Test plan:
1. Apply patch
2. Run comprehensive tests:
   $ ktd --shell
  k$ prove -v t/db_dependent/Feature_tests.t
=> SUCCESS: All TODO items resolved
3. Verify functionality works as expected
4. Sign off :-D
```

**Best Practices for TODO Management:**
1. **Track Progress**: Use ✅ COMPLETED, ⏳ IN PROGRESS, ⏳ PENDING
2. **Systematic Resolution**: Address items in logical dependency order
3. **Comprehensive Testing**: Each TODO resolution should include tests
4. **Clear Documentation**: Explain what was changed and why
5. **Follow-up Commits**: Use consistent commit message format

**Creating Database Triggers in Atomic Updates:**
```perl
use Modern::Perl;
use C4::Installer qw( trigger_exists );
use Koha::Installer::Output qw(say_warning say_success say_info);

return {
    bug_number  => "XXXXX",
    description => "Add database triggers for data integrity",
    up          => sub {
        my ($args) = @_;
        my ( $dbh, $out ) = @$args{qw(dbh out)};

        # Check if trigger exists before creating
        unless ( trigger_exists('my_trigger_name') ) {
            $dbh->do(q{
                CREATE TRIGGER my_trigger_name
                    BEFORE INSERT ON table_name
                    FOR EACH ROW
                BEGIN
                    -- Trigger logic here
                    IF condition THEN
                        SIGNAL SQLSTATE '45000'
                        SET MESSAGE_TEXT = 'Koha::Exceptions::MyException';
                    END IF;
                END
            });
            say_success( $out, "Created trigger to enforce data integrity" );
        }
    },
};
```

**Best Practices for Database Triggers:**
1. **Always check existence** using `trigger_exists()` before creation
2. **Use descriptive names** with consistent prefixes (e.g., `trg_tablename_purpose`)
3. **Include both atomic update and kohastructure.sql** for new installations
4. **Use SIGNAL SQLSTATE '45000'** for custom error messages
5. **Follow Koha exception naming** patterns in error messages
6. **Test trigger behavior** thoroughly before committing

**Testing Database Triggers:**
```bash
# Inside KTD - Check if triggers exist
ktd --shell --run "koha-mysql kohadev -e 'SHOW TRIGGERS LIKE \"table_name\";'"

# Test trigger_exists function
ktd --shell --run "cd /kohadevbox/koha && perl -MC4::Installer=trigger_exists -e 'print trigger_exists(\"trigger_name\") ? \"EXISTS\" : \"NOT FOUND\";'"

# Run atomic updates
ktd --shell --run "cd /kohadevbox/koha && perl installer/data/mysql/updatedatabase.pl"
```

**Structure and Best Practices:**

Atomic updates in Koha follow a specific structure defined in `installer/data/mysql/atomicupdate/skeleton.pl`:

```perl
use Modern::Perl;
use Koha::Installer::Output qw(say_warning say_success say_info);

return {
    bug_number  => "BUG_NUMBER",
    description => "A single line description",
    up          => sub {
        my ($args) = @_;
        my ( $dbh, $out ) = @$args{qw(dbh out)};

        # Database operations
        $dbh->do(q{});

        # Standardized output messages
        say $out "Added new system preference 'XXX'";
        say_success( $out, "Use green for success" );
        say_warning( $out, "Use yellow for warning/a call to action" );
        say_info( $out, "Use blue for further information" );
    },
};
```

**Key Requirements:**
1. **File Permissions**: Atomic update files must be executable (`chmod +x`)
2. **Output Functions**: Use `Koha::Installer::Output` for consistent messaging
3. **Standard Messages**: Follow skeleton.pl patterns for different operations:
   - Tables: "Added new table 'XXX'"
   - Columns: "Added column 'XXX.YYY'"
   - System preferences: "Added new system preference 'XXX'"
   - Permissions: "Added new permission 'XXX'"
   - Letters: "Added new letter 'XXX' (TRANSPORT)"

### Exception handling

**Creating Domain-Specific Exceptions:**

Follow the pattern established in `Koha::Exceptions::Patron`:

```perl
package Koha::Exceptions::ApiKey;

use Modern::Perl;
use Koha::Exception;

use Exception::Class (
    'Koha::Exceptions::ApiKey' => {
        isa => 'Koha::Exception',
    },
    'Koha::Exceptions::ApiKey::AlreadyRevoked' => {
        isa         => 'Koha::Exceptions::ApiKey',
        description => 'API key is already revoked'
    },
);

=head1 NAME

Koha::Exceptions::ApiKey - Base class for API key exceptions

=head1 Exceptions

=head2 Koha::Exceptions::ApiKey

Generic API key exception.

=head2 Koha::Exceptions::ApiKey::AlreadyRevoked

Exception thrown when trying to revoke an already revoked API key.

=cut

1;
```

**Best Practices:**
1. **Domain-Specific**: Create exceptions for specific domains (ApiKey, Patron, etc.)
2. **Inheritance**: Use proper ISA relationships
3. **POD Documentation**: Always include comprehensive POD documentation
4. **Semantic Naming**: Exception names should clearly indicate the problem

### System preferences

**Adding New System Preferences:**
1. **Atomic Update**: Create atomic update script for existing installations
2. **Mandatory Sysprefs**: Add to `installer/data/mysql/mandatory/sysprefs.sql` for fresh installations
3. **Alphabetical Order**: Maintain strict alphabetical order in sysprefs.sql
4. **Preference Template**: Add to appropriate `.pref` file in `koha-tmpl/intranet-tmpl/prog/en/modules/admin/preferences/`

**System Preference Structure:**
```sql
('PreferenceName', 'default_value', 'options', 'Description text', 'Type'),
```

Types: `YesNo`, `Free`, `Choice`, `Integer`, `Float`, `Textarea`

### QA tools and standards

**Common QA Issues:**
1. **File Permissions**: Atomic updates must be executable
2. **POD Coverage**: All modules need comprehensive POD documentation
3. **Alphabetical Order**: System preferences must be in correct alphabetical order
4. **HEA Considerations**: New system preferences require HEA consideration comments
5. **Forbidden Patterns**: QA tools check for specific patterns and requirements

**Running QA Tools:**
```bash
# Inside KTD
/kohadevbox/qa-test-tools/koha-qa.pl -c NUMBER_OF_COMMITS -v 2
```

**QA Tool Output Levels:**
- **PASS**: All checks passed
- **WARN**: Non-blocking warnings
- **FAIL**: Blocking issues that must be fixed
- **SKIP**: Tests skipped (usually due to missing files or conditions)

### Perl best practices

**Modern Perl Usage:**
```perl
use Modern::Perl;
use Try::Tiny qw(catch try);  # Always use Try::Tiny, never eval
use Koha::Database;
use Koha::DateUtils qw( dt_from_string );
```

**Exception Handling:**
```perl
# ✅ CORRECT - Use Try::Tiny with Koha::Logger
use Koha::Logger;

return try {
    # Code that might fail
} catch {
    # Handle exception
    Koha::Logger->get->error("Error: $_");
    return { error => 1, message => "$_" };
};

# ❌ WRONG - Don't use eval or warn
eval {
    # Code
};
if ($@) {
    warn "Error: $@";  # Don't use warn
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

### Test naming conventions

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

## Testing framework

### Test structure patterns

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

### Exception testing patterns

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

### Mocking patterns

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

### Test configuration

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

## Plugin development

### Development environment with KTD

Plugins are developed in `~/git/koha-plugins/`. Each plugin lives in its own directory. KTD mounts the plugin directory into the container.

**Example: setting up Rapido ILL for development:**
```bash
cd ~/git/koha-plugins
git clone git@github.com:bywatersolutions/koha-plugin-rapido-ill.git rapido-ill
```

**Launch a KTD instance for a single plugin:**
```bash
ktd --name rapido --proxy --single-plugin ~/git/koha-plugins/rapido-ill up -d
ktd --name rapido --wait-ready 120
```

The `--single-plugin` flag mounts only that plugin directory. Use `--plugins` instead to mount the entire `$PLUGINS_DIR` (all plugins).

**Install and test:**
```bash
# Install plugins
ktd --name rapido --shell --run "cd /kohadevbox/koha && perl misc/devel/install_plugins.pl"

# Run plugin tests
ktd --name rapido --shell --run "
  cd /kohadevbox/plugins/rapido-ill &&
  export PERL5LIB=\$PERL5LIB:Koha/Plugin/Com/ByWaterSolutions/RapidoILL/lib:. &&
  prove -v t/
"

# Restart Plack after code changes
ktd --name rapido --shell --run "koha-plack --restart kohadev"
```

**Multiple plugins:**
```bash
# Mount all plugins from $PLUGINS_DIR
ktd --name dev --proxy --plugins up -d
```

### Plugin architecture

For comprehensive understanding of Koha's plugin framework, see:
**[Koha Plugin Architecture](plugin_architecture.md)**

This guide covers:
- Plugin framework overview and core storage API
- Data persistence patterns and configuration management
- Plugin lifecycle methods (install, upgrade, uninstall)
- Configuration validation and management scripts
- Database integration with custom schema classes
- Template integration and hook system usage
- Factory methods for avoiding "Subroutine redefined" warnings
- Version management, changelog conventions, and commit format
- CI/CD with GitHub Actions and KPZ packaging
- Best practices for performance, error handling, and maintainability

## Commit standards

**Standard Commit Message Format:**
```
Bug XXXXX: Brief description of the change

Detailed explanation of what the change does and why.
Include any relevant technical details.

Test plan:
1. Step-by-step instructions for testing
2. Expected results
3. Edge cases to verify
```

**Follow-up Commits:**
```
Bug XXXXX: (follow-up) Brief description of the fix

Explanation of what QA issue or problem this addresses.
```

## Operational deployment

### System services

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

### Cron jobs

**Synchronization Scripts:**
```bash
# One entry per pod, every 5 minutes
*/5 * * * * cd /var/lib/koha/<instance>/plugins; PERL5LIB=/usr/share/koha/lib:Koha/Plugin/Com/Company/PluginName/lib:. perl Koha/Plugin/Com/Company/PluginName/scripts/sync_requests.pl --pod <pod_name>
```

**Variables:**
- `<instance>`: Koha instance name (e.g., `kohadev`, `library`)
- `<pod_name>`: Pod identifier from configuration

### Monitoring & debugging

**Log Analysis:**
```bash
# Plugin-specific logs
tail -f /var/log/koha/<instance>/plack-intranet-error.log | grep PluginName

# Task queue logs  
journalctl -u plugin-task-queue -f

# Cron job logs
grep "sync_requests" /var/log/syslog
```

**Database Trigger Debugging:**
```bash
# Inside KTD - View trigger definitions
ktd --shell --run "koha-mysql kohadev -e 'SHOW CREATE TRIGGER trigger_name;'"

# Check trigger execution errors in MySQL error log
ktd --shell --run "tail -f /var/log/mysql/error.log"

# Test trigger behavior with sample data
ktd --shell --run "koha-mysql kohadev -e 'INSERT INTO borrowers (cardnumber, userid) VALUES (\"test123\", \"test456\");'"

# Verify trigger_exists utility function
ktd --shell --run "cd /kohadevbox/koha && perl -MC4::Installer=trigger_exists -e 'print \"Trigger exists: \" . (trigger_exists(\"trg_borrowers_cardnumber_userid_insert\") ? \"YES\" : \"NO\") . \"\\n\";'"

# Test database backup includes triggers
ktd --shell --run "mysqldump -hdb -uroot -ppassword koha_kohadev > /tmp/backup.sql"
ktd --shell --run "grep -c 'CREATE.*TRIGGER' /tmp/backup.sql"

# Test trigger restoration from backup
ktd --shell --run "mysql -hdb -uroot -ppassword -e 'CREATE DATABASE test_triggers;'"
ktd --shell --run "mysql -hdb -uroot -ppassword test_triggers < /tmp/backup.sql"
ktd --shell --run "mysql -hdb -uroot -ppassword test_triggers -e 'SHOW TRIGGERS;' | wc -l"
ktd --shell --run "mysql -hdb -uroot -ppassword -e 'DROP DATABASE test_triggers;'"
```

**Configuration Validation:**
```perl
# Test configuration loading
my $plugin = Koha::Plugin::Com::Company::PluginName->new();
my $config = $plugin->configuration();
use Data::Dumper;
print Dumper($config);
```

## Best practices summary

### Development workflow
1. **Setup KTD** with proper environment variables
2. **Format code** with tidy.pl before every commit
3. **Write tests first** (TDD approach when possible)
4. **Use transactions** for all database operations
5. **Mock external dependencies** in tests
6. **Follow naming conventions** for tests and methods

### Architecture principles
1. **Separation of concerns** (Backend ↔ ActionHandlers ↔ Client)
2. **Consistent error handling** with Try::Tiny and custom exceptions
3. **Configuration-driven behavior** with YAML and defaults
4. **Transaction safety** for all database operations
5. **Comprehensive logging** for debugging and monitoring

### CSS/SCSS development

When modifying SCSS files in Koha, the compiled CSS assets must be rebuilt:

```bash
# Within KTD environment
ktd --shell --run 'cd /kohadevbox/koha && npm run css:build'
```

**Important**: 
- SCSS source files are in `koha-tmpl/*/prog/css/src/`
- Compiled CSS files are gitignored (auto-generated)
- Always rebuild after SCSS changes before testing
- Use `npm run css:build` (not yarn) to avoid gulp dependency issues

### Code quality
1. **Mandatory code formatting** with Koha standards
2. **Comprehensive test coverage** (unit + integration)
3. **Proper exception handling** throughout codebase
4. **Clear documentation** and inline comments
5. **Consistent commit messages** with issue tracking

This handbook serves as the definitive guide for Koha and plugin development, ensuring consistency, quality, and maintainability across all projects.
