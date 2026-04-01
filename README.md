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
8. [Best practices summary](#best-practices-summary)

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
ktd --shell --run "koha-mysql kohadev -e 'SELECT COUNT(*) FROM biblio;'"

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
#!/usr/bin/env perl

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

### Quality assurance

The `koha-qa.pl` script runs a suite of checks on your commits: coding standards, POD coverage, forbidden patterns, file permissions, and more.

**Running QA on Koha core patches:**
```bash
# Inside KTD — check the last N commits
ktd --shell --run "/kohadevbox/qa-test-tools/koha-qa.pl -c 2 -v 2"

# Check a specific range
ktd --shell --run "/kohadevbox/qa-test-tools/koha-qa.pl -c 5 -v 2"
```

**Running QA on plugin commits:**

Plugins live outside the Koha source tree, so you need to tell the QA tools where to find the files:

```bash
ktd --name rapido --shell --run "
  cd /kohadevbox/plugins/rapido-ill &&
  /kohadevbox/qa-test-tools/koha-qa.pl -c 1 -v 2
"
```

**Output levels:**
- **PASS**: all checks passed
- **WARN**: non-blocking warnings (review but not necessarily fix)
- **FAIL**: blocking issues that must be fixed before sign-off
- **SKIP**: checks skipped (missing files or not applicable)

**Common issues caught by QA tools:**
- Missing or incorrect GPL license headers
- POD coverage gaps in `.pm` files
- `use` of forbidden modules or patterns
- Atomic update files not executable
- System preferences not in alphabetical order
- Trailing whitespace, tabs vs spaces

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

For testing patterns, mocking, and best practices, see:
**[Koha testing framework](koha_testing_framework.md)**

This guide covers:
- Test structure patterns, file organization, and transaction rules
- TestBuilder naming conventions
- Database-dependent test templates
- Exception testing and mocking patterns
- Logger testing with `t::lib::Mocks::Logger`

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
