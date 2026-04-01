# Koha testing framework

Comprehensive guide to testing patterns, mocking, and best practices for Koha development.

## Test structure patterns

**File organization standards:**
- Database-dependent tests for `a_method` in class `Some::Class` → `t/db_dependent/Some/Class.t`
- Main subtest titled `'a_method() tests'` contains all tests for that method
- Inner subtests have descriptive titles for specific behaviors

**Standard test file structure:**
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

**Transaction rules:**
- Main subtest wrapped in transaction if only one behavior tested
- Each inner subtest wrapped in transaction if multiple behaviors tested
- Never nest transactions

**TestBuilder class naming convention:**
```perl
# IMPORTANT: TestBuilder->build_object() requires PLURAL class names
my $patron = $builder->build_object({ class => 'Koha::Patrons' });     # ✓ Correct
my $library = $builder->build_object({ class => 'Koha::Libraries' });   # ✓ Correct  
my $item = $builder->build_object({ class => 'Koha::Items' });         # ✓ Correct

# NOT singular class names
my $patron = $builder->build_object({ class => 'Koha::Patron' });      # ✗ Wrong
```

**Rule**: Always use the plural/collection class name (e.g., `Koha::Patrons`) not the singular object class name (e.g., `Koha::Patron`).

**Database-dependent test template:**
```perl
use Modern::Perl;
use Test::More tests => 2;  # use_ok + main subtest
use Test::Exception;
use Test::MockModule;
use Test::MockObject;
use Try::Tiny qw(catch try);

use t::lib::TestBuilder;
use t::lib::Mocks;
use t::lib::Mocks::Logger;

BEGIN {
    use_ok('Module::Under::Test');
}

my $schema = Koha::Database->new->schema;
my $builder = t::lib::TestBuilder->new;

subtest 'method_name() tests' => sub {
    plan tests => 3;
    
    subtest 'Successful operations' => sub {
        plan tests => 4;
        
        $schema->storage->txn_begin;
        
        my $object = $builder->build_object({ class => 'Koha::Objects' });
        
        my $result;
        lives_ok {
            $result = $object->method_under_test();
        } 'Method executes without error';
        
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

## Test naming conventions

**Subtest titles:**
- Format: `'method_name() tests'`
- Always include parentheses and use "tests" (plural)
- Examples:
  ```perl
  subtest 'item_received() tests' => sub { ... };
  subtest 'renewal_request() tests' => sub { ... };
  ```

**Test file organization:**
- Unit tests: `t/`
- Database-dependent tests: `t/db_dependent/`
- Class-based naming: `t/db_dependent/ClassName.t`

## Exception testing patterns

**Exception::Class testing:**
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

## Mocking patterns

**Mock external dependencies:**
```perl
# Mock plugin methods
my $plugin_module = Test::MockModule->new('Plugin::Class');
$plugin_module->mock('external_method', sub { return 'mocked_result'; });

# Mock objects
my $mock_client = Test::MockObject->new();
$mock_client->mock('api_call', sub { 
    my ($self, $data) = @_;
    return { success => 1 };
});
```

## Test configuration

**Disable external calls:**
```perl
# Set dev_mode in test configurations
my $config = {
    'test-pod' => {
        dev_mode => 1,  # Disables external API calls
        base_url => 'https://test.example.com',
    }
};
```

## Logger testing with t::lib::Mocks::Logger

**Setup and basic usage:**
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

**Exact message matching:**
```perl
$logger->debug_is('Debug message', 'Debug message logged correctly');
$logger->info_is('Processing item 123', 'Info message matches');
$logger->warn_is('Item not found', 'Warning message captured');
$logger->error_is('Database connection failed', 'Error logged');
```

**Regex pattern matching:**
```perl
$logger->debug_like(qr/Debug/, 'Debug message contains expected text');
$logger->info_like(qr/Processing item \d+/, 'Info message matches pattern');
$logger->warn_like(qr/Item.*not found/, 'Warning matches regex');
$logger->error_like(qr/Database.*failed/, 'Error message pattern matched');
```

**Message counting and consumption:**
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

**Method chaining and debugging:**
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

**Complete logger test example:**
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
