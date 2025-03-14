---
notion_page: https://www.notion.so/wpmedia/WordPress-Database-Custom-tables-with-BerlinDB-1b6ed22a22f08077a140cf7e751db30b?pvs=4
title: WordPress Database Custom tables with BerlinDB
---

# Managing Custom Tables in WordPress with BerlinDB

BerlinDB is a powerful database abstraction layer for WordPress that simplifies working with custom database tables. It provides a structured approach to create, query, and manage custom tables while maintaining WordPress coding standards and best practices. This guide explains how to implement custom tables using BerlinDB in WordPress plugins.

## Table of Contents

1. [Introduction to BerlinDB](#introduction-to-berlindb)
2. [Setting Up the Table Structure](#setting-up-the-table-structure)
3. [Creating Query Classes](#creating-query-classes)
4. [Working with Database Models](#working-with-database-models)
5. [Practical Usage Examples](#practical-usage-examples)
6. [Table Installation and Updates](#table-installation-and-updates)
7. [Best Practices](#best-practices)

## Introduction to BerlinDB

BerlinDB provides an object-oriented approach to database operations with several key components:

- **Table**: Defines the table schema, columns, and indexes
- **Query**: Handles database queries with methods for CRUD operations
- **Row**: Represents a single database row as an object
- **Schema**: Manages table creation and updates

This architecture separates concerns and makes database code more maintainable and testable.

## Setting Up the Table Structure

The first step is to define your table structure. This is done by creating a Table class that extends `\Dependencies\Database\Table`.

### Table Class Example

```php
<?php
namespace MyPlugin\Database\Tables;

use Dependencies\Database\Table;

/**
 * Custom table for storing cache entries
 */
class CacheTable extends Table {
    /**
     * Table name
     *
     * @var string
     */
    protected $name = 'cache_entries';

    /**
     * Database version
     *
     * @var int
     */
    protected $version = 1;

    /**
     * Setup the database schema
     *
     * @return void
     */
    protected function set_schema() {
        $this->schema = "
            id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
            url varchar(2000) NOT NULL DEFAULT '',
            type varchar(50) NOT NULL DEFAULT '',
            status varchar(20) NOT NULL DEFAULT '',
            modified datetime NOT NULL DEFAULT '0000-00-00 00:00:00',
            created datetime NOT NULL DEFAULT '0000-00-00 00:00:00',
            PRIMARY KEY (id),
            KEY url (url(191)),
            KEY type_status (type, status),
            KEY modified (modified)
        ";
    }
}
```

### Key Table Properties

- **$name**: The table name (without prefix)
- **$version**: Used for schema updates
- **set_schema()**: Defines the table columns and indexes

### About Indexes

In the schema example above, we've included several indexes:

1. **Primary Key**: `id` - Automatically indexed as the primary key
2. **Single Column Index**: `url(191)` - Indexes the first 191 characters of the URL for faster lookups
3. **Composite Index**: `type_status (type, status)` - Speeds up queries that filter by both type and status
4. **Date Index**: `modified` - Improves performance for queries that sort or filter by modification date

Indexes should be added strategically based on your most common query patterns. For example:
- Add indexes on columns frequently used in WHERE clauses
- Create composite indexes for columns often used together in queries
- Consider indexing columns used in ORDER BY statements
- Be careful not to over-index as this can slow down writes

## Creating Query Classes

Query classes handle database operations. They extend `\Dependencies\Database\Query` and define the query parameters.

### Query Class Example

```php
<?php
namespace MyPlugin\Database\Queries;

use Dependencies\Database\Query;
use MyPlugin\Database\Row\CacheRow;

/**
 * Query class for the cache_entries table
 */
class CacheQuery extends Query {
    /**
     * Name of the table
     *
     * @var string
     */
    protected $table_name = 'cache_entries';

    /**
     * String used to alias the database table
     *
     * @var string
     */
    protected $table_alias = 'ce';

    /**
     * Primary key of the database table
     *
     * @var string
     */
    protected $primary_key = 'id';

    /**
     * Row class name
     *
     * @var string
     */
    protected $item_name = CacheRow::class;

    /**
     * Timestamp fields
     *
     * @var array
     */
    protected $item_shape = [
        'id',
        'url',
        'type',
        'status',
        'modified',
        'created',
    ];

    /**
     * Fields that can be updated
     *
     * @var array
     */
    protected $item_updates = [
        'url',
        'type',
        'status',
        'modified',
    ];

    /**
     * Fields that can be used in WHERE clause
     *
     * @var array
     */
    protected $item_filters = [
        'id',
        'url',
        'type',
        'status',
        'modified',
        'created',
    ];
}
```

### Key Query Properties

- **$table_name**: The table name (without prefix)
- **$table_alias**: Short alias for use in queries
- **$primary_key**: Primary key column
- **$item_name**: Row class that represents a single record
- **$item_shape**: All columns in the table
- **$item_updates**: Columns that can be updated
- **$item_filters**: Columns that can be used in WHERE clauses

## Working with Database Models

Row classes represent individual database records as objects. They extend `\Dependencies\Database\Row`.

### Row Class Example

```php
<?php
namespace MyPlugin\Database\Row;

use Dependencies\Database\Row;

/**
 * Row class for the cache_entries table
 */
class CacheRow extends Row {
    /**
     * Cache entry constructor
     *
     * @param object $item Current row details.
     */
    public function __construct( $item ) {
        parent::__construct( $item );
        
        // Optional: Parse or format specific properties
        if ( ! empty( $this->modified ) && '0000-00-00 00:00:00' !== $this->modified ) {
            $this->modified = mysql2date( 'U', $this->modified, false );
        }
        
        if ( ! empty( $this->created ) && '0000-00-00 00:00:00' !== $this->created ) {
            $this->created = mysql2date( 'U', $this->created, false );
        }
    }
    
    /**
     * Check if the entry is pending
     *
     * @return bool
     */
    public function is_pending() {
        return 'pending' === $this->status;
    }
    
    /**
     * Check if the entry is completed
     *
     * @return bool
     */
    public function is_completed() {
        return 'completed' === $this->status;
    }
    
    /**
     * Get entry age in seconds
     *
     * @return int
     */
    public function get_age() {
        return time() - $this->created;
    }
}
```

Row classes can include:
- Custom constructors for data formatting
- Helper methods for business logic
- Computed properties

## Practical Usage Examples

### Initializing the Database Components

```php
<?php
namespace MyPlugin;

use MyPlugin\Database\Tables\CacheTable;
use MyPlugin\Database\Queries\CacheQuery;

/**
 * Database Manager
 */
class Database {
    /**
     * Table instance
     *
     * @var CacheTable
     */
    private $table;
    
    /**
     * Query instance
     *
     * @var CacheQuery
     */
    private $query;
    
    /**
     * Constructor
     */
    public function __construct() {
        $this->table = new CacheTable();
        $this->query = new CacheQuery();
    }
    
    /**
     * Get the query instance
     *
     * @return CacheQuery
     */
    public function get_query() {
        return $this->query;
    }
    
    /**
     * Get the table instance
     *
     * @return CacheTable
     */
    public function get_table() {
        return $this->table;
    }
}
```

### Basic CRUD Operations

```php
<?php
// Create a new record
$item_id = $query->add_item([
    'url'      => 'https://example.com/page',
    'type'     => 'page',
    'status'   => 'pending',
    'modified' => current_time('mysql'),
    'created'  => current_time('mysql'),
]);

// Get a single item by ID
$item = $query->get_item(123);

// Get items with filtering
$items = $query->query([
    'status'  => 'pending',
    'orderby' => 'created',
    'order'   => 'DESC',
    'number'  => 50,
    'offset'  => 0,
]);

// Update an item
$updated = $query->update_item(
    123,
    [
        'status'   => 'completed',
        'modified' => current_time('mysql'),
    ]
);

// Delete an item
$deleted = $query->delete_item(123);

// Count items
$count = $query->count([
    'status' => 'pending',
]);
```

### Advanced Queries

```php
<?php
// Query with multiple conditions
$items = $query->query([
    'status__in'    => ['pending', 'processing'],
    'created__gte'  => date('Y-m-d H:i:s', strtotime('-1 day')),
    'orderby'       => 'id',
    'order'         => 'ASC',
    'number'        => 100,
]);

// Query with custom WHERE clause
$items = $query->query([
    'status' => 'pending',
    'where'  => [
        'column'  => 'url',
        'value'   => 'example.com',
        'compare' => 'LIKE',
    ],
]);

// Query with multiple custom WHERE clauses
$items = $query->query([
    'where' => [
        'relation' => 'AND',
        [
            'column'  => 'status',
            'value'   => 'pending',
            'compare' => '=',
        ],
        [
            'column'  => 'type',
            'value'   => 'page',
            'compare' => '=',
        ],
    ],
]);
```

## Table Installation and Updates

BerlinDB handles table creation and updates through the Table class. You need to call the `install()` method during plugin activation.

### Table Installation

```php
<?php
/**
 * Plugin activation hook
 */
function my_plugin_activate() {
    $database = new \MyPlugin\Database();
    $table = $database->get_table();
    
    // Create or update the table
    $table->install();
}
register_activation_hook( MY_PLUGIN_FILE, 'my_plugin_activate' );
```

### Schema Updates

When you need to update your table schema:

1. Increment the `$version` property in your Table class
2. Update the `set_schema()` method with the new schema
3. The table will be updated automatically during plugin activation

## Best Practices

### 1. Organize Files in a Consistent Structure

```
my-plugin/
├── inc/
│   └── Database/
│       ├── Queries/
│       │   └── CacheQuery.php
│       ├── Row/
│       │   └── CacheRow.php
│       ├── Tables/
│       │   └── CacheTable.php
│       └── Database.php
```

### 2. Use Dependency Injection

Inject database components rather than creating them directly:

```php
<?php
class CacheManager {
    private $query;
    
    public function __construct( CacheQuery $query ) {
        $this->query = $query;
    }
}
```

### 3. Add Table Prefixes Properly

BerlinDB automatically adds the WordPress table prefix, so don't include it in your table name.

### 4. Handle Timestamps Consistently

```php
<?php
// When adding/updating records
$data = [
    'url'      => 'https://example.com',
    'type'     => 'page',
    'status'   => 'pending',
    'modified' => current_time('mysql'),
    'created'  => current_time('mysql'),
];

// When retrieving records, format in Row class
public function __construct( $item ) {
    parent::__construct( $item );
    
    // Convert MySQL datetime to Unix timestamp
    if ( ! empty( $this->created ) && '0000-00-00 00:00:00' !== $this->created ) {
        $this->created = mysql2date( 'U', $this->created, false );
    }
}
```

### 5. Design Indexes for Query Patterns

Create indexes based on how you'll query the data:

```php
// Composite index for queries that filter by both type and status
$items = $query->query([
    'type'   => 'page',
    'status' => 'pending',
]);

// Single column index for sorting by modification date
$items = $query->query([
    'orderby' => 'modified',
    'order'   => 'DESC',
]);
```

### 6. Check for Table Existence

```php
<?php
function my_plugin_init() {
    $database = new \MyPlugin\Database();
    $table = $database->get_table();
    
    // Check if table exists before using it
    if ( ! $table->exists() ) {
        $table->install();
    }
}
add_action( 'plugins_loaded', 'my_plugin_init' );
```

### 7. Use Transactions for Multiple Operations

```php
<?php
global $wpdb;

// Start transaction
$wpdb->query('START TRANSACTION');

try {
    // Add main record
    $item_id = $query->add_item([
        'url'      => 'https://example.com/page',
        'type'     => 'page',
        'status'   => 'pending',
        'modified' => current_time('mysql'),
        'created'  => current_time('mysql'),
    ]);
    
    // Add related records or perform other operations
    // ...
    
    // Commit if successful
    $wpdb->query('COMMIT');
} catch (\Exception $e) {
    // Rollback on error
    $wpdb->query('ROLLBACK');
    error_log('Transaction failed: ' . $e->getMessage());
}
```

## Conclusion

BerlinDB provides a robust framework for working with custom tables in WordPress. By following the patterns outlined in this guide, you can create maintainable, efficient database operations that follow WordPress best practices.

The separation of concerns between Table, Query, and Row classes makes your code more testable and easier to understand. This approach is particularly valuable for plugins that need to store and retrieve large amounts of custom data efficiently.