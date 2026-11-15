# WordPress Plugin Lifecycle Diagnostics

A practical guide to diagnosing WordPress plugin lifecycle issues, activation failures, dependency conflicts, hook execution problems, and plugin loading errors.

## Table of Contents

- [Introduction](#introduction)
- [Understanding the Plugin Lifecycle](#understanding-the-plugin-lifecycle)
- [Plugin Activation Failures](#plugin-activation-failures)
- [Activation and Deactivation Hooks](#activation-and-deactivation-hooks)
- [Uninstall Problems](#uninstall-problems)
- [Plugin Dependency Conflicts](#plugin-dependency-conflicts)
- [Must-Use Plugins](#must-use-plugins)
- [Plugin Loading and Initialization](#plugin-loading-and-initialization)
- [WP-CLI Diagnostics](#wp-cli-diagnostics)
- [Common Troubleshooting Scenarios](#common-troubleshooting-scenarios)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [References](#references)

## Introduction

WordPress plugins extend core functionality through PHP files, hooks, filters, database operations, and external dependencies.

A plugin lifecycle problem can cause:

- Fatal errors during activation.
- Missing plugin functionality.
- Failed deactivation routines.
- Incomplete uninstall operations.
- Plugin dependency errors.
- Unexpected behavior after updates.
- Conflicts between plugins.

This repository provides a practical approach to investigating these problems.

## Understanding the Plugin Lifecycle

A typical WordPress plugin follows these stages:

1. Plugin installation.
2. Plugin loading.
3. Plugin activation.
4. Plugin initialization.
5. Plugin execution.
6. Plugin deactivation.
7. Plugin uninstall.

Each stage has different responsibilities.

| Stage | Main responsibility |
|---|---|
| Installation | Place plugin files in the plugins directory |
| Loading | Include active plugin files |
| Activation | Perform initial setup |
| Initialization | Register hooks and functionality |
| Deactivation | Remove temporary runtime data |
| Uninstall | Remove plugin-specific permanent data |

Understanding these stages helps identify where a failure occurs.

## Plugin Activation Failures

A plugin may fail to activate because of:

- PHP syntax errors.
- Missing PHP extensions.
- Unsupported PHP versions.
- Missing required plugins.
- Undefined functions or classes.
- Incorrect file paths.
- Database errors.
- Memory limitations.

### Inspect PHP errors

Enable debugging in `wp-config.php` during development:

```php
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
define( 'WP_DEBUG_DISPLAY', false );
```

Check:

```text
wp-content/debug.log
```

Also inspect the PHP error log provided by the hosting environment.

Do not expose debugging output on a production website.

### Verify plugin files

Check whether the main plugin file exists and contains a valid plugin header.

```php
$plugin_file = WP_PLUGIN_DIR . '/example-plugin/example-plugin.php';

if ( file_exists( $plugin_file ) ) {
    error_log( 'Plugin file exists.' );
} else {
    error_log( 'Plugin file is missing.' );
}
```

## Activation and Deactivation Hooks

WordPress provides dedicated hooks for plugin activation and deactivation.

### Activation hook

Use `register_activation_hook()` to perform setup operations.

```php
function example_plugin_activate() {
    add_option( 'example_plugin_version', '1.0.0' );
}

register_activation_hook(
    __FILE__,
    'example_plugin_activate'
);
```

Typical activation tasks include:

- Creating default options.
- Registering initial database structures.
- Setting up required configuration.
- Registering custom post types before flushing rewrite rules.

### Deactivation hook

Use `register_deactivation_hook()` for temporary cleanup.

```php
function example_plugin_deactivate() {
    wp_clear_scheduled_hook( 'example_plugin_daily_task' );
}

register_deactivation_hook(
    __FILE__,
    'example_plugin_deactivate'
);
```

Do not use deactivation hooks to delete all permanent plugin data unless that behavior is explicitly intended.

## Uninstall Problems

Uninstalling a plugin is different from deactivating it.

A proper uninstall routine should remove plugin-specific data when the administrator chooses to delete the plugin.

### Using uninstall.php

Create an `uninstall.php` file in the plugin root directory.

```php
<?php

if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

delete_option( 'example_plugin_settings' );
delete_transient( 'example_plugin_cache' );
```

For Multisite, consider site-specific options and network-wide data separately.

### Important distinction

| Operation | Recommended responsibility |
|---|---|
| Deactivation | Remove temporary data and scheduled tasks |
| Uninstall | Remove permanent plugin data when appropriate |

Always verify the expected cleanup behavior before deleting database tables or user data.

## Plugin Dependency Conflicts

Some plugins depend on other plugins, PHP extensions, or specific WordPress features.

Common problems include:

- Required plugin is inactive.
- Required class is unavailable.
- Dependency version is incompatible.
- Plugin load order is incorrect.
- Required PHP extension is missing.

### Check whether a plugin is active

```php
require_once ABSPATH . 'wp-admin/includes/plugin.php';

if ( is_plugin_active( 'woocommerce/woocommerce.php' ) ) {
    error_log( 'WooCommerce is active.' );
} else {
    error_log( 'WooCommerce is not active.' );
}
```

### Check for a required class

```php
if ( class_exists( 'WooCommerce' ) ) {
    error_log( 'WooCommerce class is available.' );
} else {
    error_log( 'WooCommerce class is unavailable.' );
}
```

Avoid assuming that a dependency is available before its code has loaded.

## Must-Use Plugins

Must-Use Plugins, commonly called MU-Plugins, are automatically loaded from:

```text
wp-content/mu-plugins/
```

They are not managed like ordinary plugins.

Important characteristics:

- Automatically enabled.
- Cannot be deactivated from the standard Plugins screen.
- Loaded before regular plugins.
- Do not use normal activation and deactivation hooks.
- Can contain hosting-specific functionality.

### Inspect MU-Plugins

Using WP-CLI:

```bash
wp plugin list --status=must-use
```

Check the directory manually if the command or environment does not provide the expected information.

A broken MU-Plugin can affect the entire WordPress installation. Inspect this directory when ordinary plugin deactivation does not resolve an issue.

## Plugin Loading and Initialization

Plugin code should be initialized at the appropriate WordPress lifecycle stage.

Example:

```php
add_action( 'plugins_loaded', 'example_plugin_initialize' );

function example_plugin_initialize() {
    if ( ! class_exists( 'WooCommerce' ) ) {
        return;
    }

    // Initialize WooCommerce integration.
}
```

Common initialization problems include:

- Calling functions before WordPress loads them.
- Registering hooks too late.
- Accessing dependency classes too early.
- Loading files from incorrect paths.
- Executing admin-only code on frontend requests.

Use appropriate hooks and verify the execution context.

## WP-CLI Diagnostics

WP-CLI provides useful commands for inspecting plugin status and lifecycle operations.

### List installed plugins

```bash
wp plugin list
```

### Check plugin status

```bash
wp plugin status
```

### Activate a plugin

```bash
wp plugin activate example-plugin
```

### Deactivate a plugin

```bash
wp plugin deactivate example-plugin
```

### Check whether a plugin is active

```bash
wp plugin is-active example-plugin
```

### Verify plugin checksums

```bash
wp plugin verify-checksums example-plugin
```

Note: Checksum verification is intended for plugins with available WordPress.org checksums. It may not apply to premium or custom plugins.

### Inspect plugin details

```bash
wp plugin get example-plugin
```

Use these commands carefully on production websites. Activation and deactivation can change website behavior.

## Common Troubleshooting Scenarios

### Scenario 1: Plugin activation causes a fatal error

Investigate:

1. PHP error logs.
2. Plugin file integrity.
3. PHP compatibility.
4. Missing dependencies.
5. Conflicting plugins.

### Scenario 2: Plugin functionality disappears after activation

Investigate:

1. Whether the plugin is actually active.
2. Whether initialization hooks execute.
3. Whether required dependencies are loaded.
4. Whether PHP errors interrupt execution.

### Scenario 3: Plugin data remains after deletion

Investigate:

1. Whether an uninstall routine exists.
2. Whether `uninstall.php` is correctly located.
3. Whether the cleanup routine executes.
4. Whether data is stored in site options or network options.

### Scenario 4: Plugin cannot be disabled normally

Investigate:

1. Whether it is an MU-Plugin.
2. Whether it is network activated.
3. Whether the current user has sufficient permissions.
4. Whether another management system controls activation.

## Security Best Practices

1. Never modify WordPress core files to fix plugin problems.
2. Validate user input and escape output.
3. Check user capabilities before administrative operations.
4. Use nonces for administrative actions.
5. Avoid deleting user data during ordinary deactivation.
6. Protect uninstall routines against direct access.
7. Do not expose PHP errors publicly.
8. Back up the database before destructive cleanup.
9. Test plugin lifecycle operations in a staging environment.
10. Keep MU-Plugins documented and regularly reviewed.

## Troubleshooting Checklist

- [ ] Confirm the plugin is installed.
- [ ] Verify the plugin header.
- [ ] Check PHP compatibility.
- [ ] Review PHP error logs.
- [ ] Verify required dependencies.
- [ ] Check activation and deactivation hooks.
- [ ] Inspect uninstall routines.
- [ ] Review MU-Plugins.
- [ ] Check plugin loading order.
- [ ] Run relevant WP-CLI diagnostics.
- [ ] Test for plugin conflicts.
- [ ] Verify database cleanup behavior.

## References

- [WordPress Plugin Developer Handbook](https://developer.wordpress.org/plugins/)
- [Activation and Deactivation Hooks](https://developer.wordpress.org/plugins/plugin-basics/activation-deactivation-hooks/)
- [Uninstall Methods](https://developer.wordpress.org/plugins/plugin-basics/uninstall-methods/)
- [Must-Use Plugins](https://developer.wordpress.org/advanced-administration/plugins/mu-plugins/)
- [WP-CLI Plugin Commands](https://developer.wordpress.org/cli/commands/plugin/)

## License

This repository is intended for educational and technical reference purposes.
