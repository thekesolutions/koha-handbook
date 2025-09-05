# Koha Template::Toolkit System Architecture

## Overview

Koha uses Template::Toolkit (TT) as its templating engine, providing a sophisticated multi-language, multi-theme system. The architecture centers around `C4::Templates` and `C4::Languages`, enabling dynamic template resolution based on interface, theme, and language preferences.

## Core Architecture Components

### 1. C4::Templates - Template Management Engine

The `C4::Templates` module serves as the primary interface for template processing:

```perl
# Template instantiation
my $template = C4::Templates->new($interface, $filename, $tmplbase, $query);

# Key parameters:
# $interface - 'intranet' or 'opac'
# $filename  - template file name (e.g., 'admin-home.tt')
# $tmplbase  - base template path
# $query     - CGI query object for language/theme detection
```

### 2. C4::Languages - Language Resolution

Handles language detection and management:

```perl
use C4::Languages qw( getlanguage getTranslatedLanguages get_bidi );

# Language detection from multiple sources
my $lang = C4::Languages::getlanguage($query);
# Sources: URL parameter, cookie, browser Accept-Language, system preference
```

### 3. Template Directory Structure

```
koha-tmpl/
├── intranet-tmpl/
│   └── prog/                    # Default intranet theme
│       ├── en/                  # English templates
│       │   ├── modules/         # Page templates
│       │   ├── includes/        # Reusable components
│       │   ├── data/            # Static data files
│       │   └── xslt/           # XSLT stylesheets
│       ├── es-ES/              # Spanish templates
│       └── fr-FR/              # French templates
└── opac-tmpl/
    └── bootstrap/               # Default OPAC theme
        ├── en/
        ├── es-ES/
        └── fr-FR/
```

## Template Resolution Process

### 1. Theme and Language Detection

The `themelanguage()` function orchestrates template resolution:

```perl
sub themelanguage {
    my ( $htdocs, $tmpl, $interface, $query ) = @_;
    
    # 1. Detect language from query, cookie, browser, or syspref
    my $lang = C4::Languages::getlanguage($query);
    
    # 2. Determine available themes for interface
    return availablethemes( $htdocs, $tmpl, $interface, $lang );
}
```

### 2. Template Path Resolution

```perl
# Path construction in C4::Templates->new()
my $htdocs = ($interface eq "intranet") 
    ? C4::Context->config('intrahtdocs')    # /koha-tmpl/intranet-tmpl
    : C4::Context->config('opachtdocs');    # /koha-tmpl/opac-tmpl

# Include paths for template inheritance
my @includes;
foreach my $theme (@$activethemes) {
    push @includes, "$htdocs/$theme/$lang/includes";
    push @includes, "$htdocs/$theme/en/includes" unless $lang eq 'en';
}
```

### 3. Fallback Mechanism

Koha implements a sophisticated fallback system:

1. **Language Fallback**: `$theme/$lang/` → `$theme/en/`
2. **Theme Fallback**: Primary theme → Fallback themes
3. **Plugin Integration**: Plugin template paths added to includes

## Template::Toolkit Configuration

### Template Object Creation

```perl
my $template = Template->new({
    INCLUDE_PATH   => \@includes,           # Template search paths
    ABSOLUTE       => 1,                    # Allow absolute paths
    PLUGIN_BASE    => 'Koha::Template::Plugin', # Custom plugins
    COMPILE_EXT    => '.ttc',               # Compiled template extension
    COMPILE_DIR    => $template_cache_dir,  # Template compilation cache
    FILTERS        => {
        'html'     => \&html_filter,        # HTML escaping
        'url'      => \&url_filter,         # URL encoding
        'json'     => \&json_filter,        # JSON encoding
    },
    ENCODING       => 'UTF-8',              # Character encoding
});
```

### Template Caching

```perl
# Template caching configuration
my $use_template_cache = C4::Context->config('template_cache_dir') 
                      && defined $ENV{GATEWAY_INTERFACE};

# Cache disabled for command-line scripts
# Enabled for web requests when template_cache_dir is configured
```

## Template Syntax and Patterns

### 1. Basic Template Structure

```html
[% USE raw %]
[% USE Koha %]
[% PROCESS 'i18n.inc' %]
[% SET footerjs = 1 %]
[% INCLUDE 'doc-head-open.inc' %]

<title>
    [% FILTER collapse %]
        [% t("Page Title") | html %]
        &rsaquo; [% t("Koha") | html %]
    [% END %]
</title>

[% INCLUDE 'doc-head-close.inc' %]
</head>

<body id="module_page" class="module">
    [% WRAPPER 'header.inc' %]
        [% INCLUDE 'breadcrumbs.inc' %]
    [% END %]
    
    [% WRAPPER 'main-container.inc' %]
        <!-- Page content -->
    [% END %]
    
    [% INCLUDE 'intranet-bottom.inc' %]
</body>
</html>
```

### 2. Internationalization (i18n)

```html
<!-- Translation function -->
[% t("Text to translate") %]

<!-- With HTML escaping -->
[% t("Text to translate") | html %]

<!-- With parameters -->
[% t("Hello [_1]", username) %]

<!-- Pluralization -->
[% tn("1 item", "[_1] items", count, count) %]
```

### 3. Common Template Patterns

```html
<!-- Conditional rendering -->
[% IF condition %]
    Content when true
[% ELSIF other_condition %]
    Alternative content
[% ELSE %]
    Default content
[% END %]

<!-- Loops -->
[% FOREACH item IN items %]
    <li>[% item.name | html %]</li>
[% END %]

<!-- Includes with parameters -->
[% INCLUDE 'component.inc' 
    param1 = value1
    param2 = value2 %]

<!-- Wrappers for layout -->
[% WRAPPER 'layout.inc' %]
    Content to be wrapped
[% END %]
```

### 4. Koha-Specific Plugins and Filters

```html
<!-- Koha system preferences -->
[% IF Koha.Preference('SystemPreference') %]
    <!-- Content based on syspref -->
[% END %]

<!-- User permissions -->
[% IF CAN_user_module_permission %]
    <!-- Content for authorized users -->
[% END %]

<!-- Asset URLs -->
[% INCLUDE 'css.inc' %]
[% INCLUDE 'js.inc' %]

<!-- Data formatting -->
[% date | $KohaDates %]
[% price | $Price %]
```

## Language and Localization System

### 1. Language Detection Priority

1. **URL Parameter**: `?language=es-ES`
2. **Cookie**: `KohaOpacLanguage` or `KohaIntranetLanguage`
3. **Browser**: `Accept-Language` header
4. **System Default**: `language` system preference

### 2. Translation File Structure

```
misc/translator/po/
├── en/                          # English (source)
├── es-ES/                       # Spanish
│   ├── intranet.po             # Staff interface translations
│   ├── opac.po                 # OPAC translations
│   └── installer.po            # Installer translations
└── fr-FR/                       # French
```

### 3. Translation Workflow

```bash
# Extract translatable strings
perl misc/translator/translate extract es-ES

# Update translation files
perl misc/translator/translate update es-ES

# Install translations
perl misc/translator/translate install es-ES
```

## Plugin Integration

### 1. Template Include Paths

Plugins can extend template search paths:

```perl
# In plugin
sub template_include_paths {
    my ($self, $params) = @_;
    my $interface = $params->{interface};
    my $lang = $params->{lang};
    
    return [
        "$plugin_dir/templates/$interface/$lang",
        "$plugin_dir/templates/$interface/en"
    ];
}
```

### 2. Custom Template Variables

```perl
# In plugin hook
sub intranet_head {
    my ($self) = @_;
    my $template = $self->get_template({ file => 'head.tt' });
    
    $template->param(
        plugin_var => $self->get_plugin_data(),
        custom_css => $self->get_css_url(),
    );
    
    return $template->output;
}
```

## Performance Considerations

### 1. Template Compilation Caching

```perl
# Enable template caching in koha-conf.xml
<config>
    <template_cache_dir>/tmp/koha_cache</template_cache_dir>
</config>

# Templates compiled to .ttc files for faster loading
# Cache automatically invalidated when source templates change
```

### 2. Include Path Optimization

```perl
# Minimize include paths for better performance
# Order paths by likelihood of template location
# Use specific paths rather than broad wildcards
```

### 3. Memory Usage

```perl
# Template objects cached in memory
# Use Koha::Cache::Memory::Lite for template metadata
# Avoid creating unnecessary template objects
```

## Development Best Practices

### 1. Template Organization

```html
<!-- Use semantic file naming -->
admin-home.tt              # Main admin page
admin-preferences.tt       # Preferences page
includes/admin-menu.inc    # Reusable admin menu

<!-- Consistent directory structure -->
modules/                   # Main page templates
includes/                  # Reusable components
data/                     # Static data files
```

### 2. Security Considerations

```html
<!-- Always escape user data -->
[% user_input | html %]

<!-- Use appropriate filters -->
[% url_param | url %]
[% json_data | json %]

<!-- Validate template paths -->
# C4::Templates::badtemplatecheck() prevents path traversal
```

### 3. Accessibility and Standards

```html
<!-- Use semantic HTML -->
[% WRAPPER breadcrumbs %]
    [% WRAPPER breadcrumb_item bc_active=1 %]
        <span>Current Page</span>
    [% END %]
[% END %]

<!-- Include ARIA attributes -->
<button aria-label="[% t('Close dialog') %]">×</button>

<!-- Proper heading hierarchy -->
<h1>[% t("Main Title") %]</h1>
<h2>[% t("Section Title") %]</h2>
```

## Debugging and Development Tools

### 1. Template Debugging

```perl
# Enable TT debugging
$ENV{TEMPLATE_DEBUG} = 1;

# Template path resolution debugging
warn "Template paths: " . join(", ", @includes);

# Variable inspection in templates
[% USE Dumper %]
[% Dumper.dump(variable_name) %]
```

### 2. Language Testing

```bash
# Test different languages
http://koha.local/cgi-bin/koha/admin/admin-home.pl?language=es-ES

# Check translation coverage
perl misc/translator/translate check es-ES
```

### 3. Template Validation

```bash
# Check template syntax
perl -c template_file.tt

# Validate HTML output
# Use browser developer tools or HTML validators
```

## Common Patterns and Examples

### 1. Data Tables

```html
[% WRAPPER 'datatables.inc' %]
<table id="data-table">
    <thead>
        <tr>
            <th>[% t("Column 1") %]</th>
            <th>[% t("Column 2") %]</th>
        </tr>
    </thead>
    <tbody>
        [% FOREACH row IN data %]
        <tr>
            <td>[% row.field1 | html %]</td>
            <td>[% row.field2 | html %]</td>
        </tr>
        [% END %]
    </tbody>
</table>
[% END %]
```

### 2. Form Handling

```html
<form method="post" action="/cgi-bin/koha/admin/process.pl">
    [% INCLUDE 'csrf-token.inc' %]
    
    <fieldset>
        <legend>[% t("Form Section") %]</legend>
        
        <label for="field1">[% t("Field Label") %]:</label>
        <input type="text" id="field1" name="field1" 
               value="[% field1 | html %]" required>
        
        [% IF errors.field1 %]
            <span class="error">[% errors.field1 | html %]</span>
        [% END %]
    </fieldset>
    
    <button type="submit">[% t("Submit") %]</button>
</form>
```

### 3. Modal Dialogs

```html
[% WRAPPER 'modal.inc' modal_id="confirm-dialog" %]
    <h3>[% t("Confirm Action") %]</h3>
    <p>[% t("Are you sure you want to proceed?") %]</p>
    
    <button class="btn btn-primary" data-action="confirm">
        [% t("Yes, proceed") %]
    </button>
    <button class="btn btn-secondary" data-dismiss="modal">
        [% t("Cancel") %]
    </button>
[% END %]
```

This Template::Toolkit system provides Koha with a powerful, flexible, and maintainable approach to generating multilingual, themed user interfaces while maintaining clean separation between presentation logic and business logic.
