# Koha Community Coding Guidelines

Source: https://wiki.koha-community.org/wiki/Coding_Guidelines (last synced 2026-05-08)

This is a local copy for coding agents. The wiki is the authoritative source.

## Basics

- If you fix code that violates guidelines, QA can still pass. You don't have to fix everything unrelated to your patch.
- Refactoring unrelated to your fix goes in a separate patch.
- Every commit must reference a bug number from bugs.koha-community.org.
- Indentation: 4 spaces (not tabs).

### AI and LLM-assisted contributions

- A human contributor remains the responsible author for every commit.
- Use of AI tools must be declared with: `Assisted-by: <ModelName> <Version> (<Vendor>)`
- Routine autocompletion/linting does not require disclosure.
- Code undergoes the same sign-off and QA process as human-written code.

## Perl

### PERL1: Perltidy

Code must be tidy using perltidy and the `.perltidyrc` in the Koha root.

### PERL2: Modern::Perl

All scripts and modules must use `use Modern::Perl;`

### PERL3: Fix warnings

Fix the code, don't add `no warnings`.

### PERL4: Perlcritic

All Perl must validate perlcritic level 5.

### PERL9: Subroutine naming conventions

**Koha namespace (current):**
- Subroutines: `snake_case`
- Module names: `UpperCamelCase`
- Variable names: `snake_case`

**C4 namespace (outdated):**
- VerbName pattern: `AddBiblio`, `GetBranch`, `ModReserve`, `DelItem`
- Singular for single object, plural for multiple
- Internal functions: `_underscore_lowercase`

### PERL13: POD

Every module needs: NAME, SYNOPSIS, DESCRIPTION, FUNCTIONS, AUTHOR sections.
Every subroutine must include POD with usage example and description.

### PERL14: Exports

Exports should be minimal. Ideally nothing exported from Koha:: namespace.

### PERL15: Object-oriented code and Koha:: namespace

- Code in Koha:: should be OO when it makes sense.
- Koha:: must not reference C4:: (except C4::Context).
- Use relationship accessors that allow prefetch (use `_result->relationship` not `Koha::Objects->find`).

### PERL16: Hashrefs for arguments

Subroutines with multiple arguments must use hashrefs:

```perl
sub myroutine {
    my ($args) = @_;
    return $args->{index} . ' => ' . $args->{value};
}
myroutine({ index => 'kw', value => 'smith' });
```

### PERL17: Unit tests required

- Tests required for ALL new routines and changes to existing routines.
- `t/` = database independent; `t/db_dependent/` = needs Koha DB.
- Don't rely on sample data; use TestBuilder.
- Subtests should be used.

### PERL20: Koha namespace

Modules should be OO using Koha::Object(s) as preferred base.

### PERL22: Plack friendly coding

Global state variables in modules and CGI scripts are forbidden.

### PERL23: Single argument

Methods take a single argument: database ID, arrayref, or hashref.

### PERL25: Read only variables

Use `use constant`, not Readonly.

### PERL26: Koha::Exceptions

Use `Koha::Exceptions` instead of die/croak. Prefer existing exception classes (MissingParameter, WrongParameter) before creating new ones.

### PERL29: Direct Object Notation

Use `CGI->new` not `new CGI`.

### PERL31: Prefer "use" over "require"

`use` is preferred. `require` only for: runtime-determined module names, or fixing circular dependencies.

## Database

### SQL7: Primary keys

New tables must have a primary key named `tablename_id`.

### SQL8: No SQL in CGI scripts

SQL belongs in C4/*.pm or Koha/*.pm modules. Exception: atomic update scripts (SQL only, no Koha module calls).

### SQL9: SQL formatting

- Reserved words in CAPITALS
- 2-space indent for FROM, WHERE, ORDER BY, GROUP BY, LIMIT
- 2 more spaces for JOINs and ANDs

### SQL10: Placeholders

Always use placeholders to prevent SQL injection.

### SQL11: Document fields in kohastructure.sql

Add COMMENT to every column definition.

### SQL12: Booleans

Use `TINYINT(1)` in DB. Annotate in DBIC schema with `is_boolean => 1`.

### SQL13: Modifying columns with FK constraints

Temporarily drop FK, modify column, re-add FK.

## Security

### SEC1: CSRF protection

- CSRF token required for all POST/PUT/DELETE/PATCH forms.
- Token via `[% INCLUDE 'csrf-token.inc' %]` in templates.
- JavaScript: `$('meta[name="csrf-token"]').attr('content')`
- Stateless requests (GET) must not have 'op' starting with 'cud-'.

## JavaScript

### JS1: Embedded scripts need CSP nonce

```html
<script nonce="[% Koha.CSPNonce | $raw %]">
```

### JS8: ESLint

JavaScript should conform to ESLint guidelines.

### JS14: Prettier

New JS files must be tidied with Prettier via `perl misc/devel/tidy.pl path/file.js`.

### JS15: JSDoc

All new JS files must include JSDoc comments for classes and functions.

### JS19: No TT tags in script tags

TT variables must be in a separate `<script>` block from the main JS logic.

## Templates

### HTML9: Filter all template variables

- `[% var | html %]` for display
- `[% var | uri %]` for URL query strings
- `[% var | $raw %]` only when explicitly safe

## Deprecations

### DEPR3: C4 deprecated for new modules

New modules go in the Koha:: namespace. C4:: is deprecated.

## Scripts

### CMD1: Use Koha::Script base class

All new command-line scripts must `use Koha::Script` or `use Koha::Script -cron`.
