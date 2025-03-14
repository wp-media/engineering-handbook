---
notion_page: https://www.notion.so/wpmedia/WordPress-Database-Queries-Best-Practices-1b6ed22a22f0805b82c5f456ef66ebc5?pvs=4
title: WordPress Database Queries: Best Practices
---

# WordPress Database Queries: Best Practices

WordPress provides a robust database abstraction layer through its global `$wpdb` object. Using this API correctly is essential for building performant, secure, and maintainable plugins and themes. This guide covers best practices for handling database queries in WordPress projects.

## Table of Contents

1. [Security: Prepared Statements](#security-prepared-statements)
2. [Common Mistakes with Dynamic Queries](#common-mistakes-with-dynamic-queries)
3. [Performance: Caching Database Results](#performance-caching-database-results)
4. [PHPCS Rules and Annotations](#phpcs-rules-and-annotations)
5. [Advanced Techniques](#advanced-techniques)

> **Note:** For basic WordPress database API usage, refer to the [official WordPress documentation on the $wpdb class](https://developer.wordpress.org/reference/classes/wpdb/).

## Security: Prepared Statements

### Why Use Prepared Statements?

Prepared statements protect against SQL injection by separating SQL code from data. **Always** use prepared statements when incorporating any variable data in your queries.

### Using `$wpdb->prepare()`

```php
global $wpdb;

$user_id = $_GET['user_id']; // Potentially unsafe user input

// WRONG - SQL injection vulnerability:
$results = $wpdb->get_results( "SELECT * FROM $wpdb->users WHERE ID = $user_id" );

// RIGHT - Using prepared statements:
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM $wpdb->users WHERE ID = %d",
        $user_id
    )
);
```

### Placeholder Types

WordPress supports these placeholders:

- `%s` - String (most common, can be used for most data types)
- `%d` - Integer
- `%f` - Float
- `%%` - Literal percentage character

```php
$wpdb->prepare(
    "SELECT * FROM $wpdb->posts WHERE post_type = %s AND post_status = %s AND ID > %d",
    $post_type,
    $post_status,
    $min_id
);
```

## Common Mistakes with Dynamic Queries

### Table/Column Names in Prepared Statements

A common mistake is using prepared statements for table or column names. **This doesn't work correctly** because `$wpdb->prepare()` will add quotes around the values, which breaks SQL syntax for identifiers.

#### Incorrect Approach

```php
// WRONG - This will cause syntax errors
$column = 'post_title';
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT %s FROM $wpdb->posts",
        $column
    )
);
// Results in: SELECT 'post_title' FROM wp_posts (invalid SQL)
```

#### Correct Approach

For dynamic table or column names:

1. Sanitize the identifiers
2. Use backticks for column/table names
3. Insert them directly into the query string (not via prepared statements)

```php
// RIGHT - Sanitize and use directly with backticks
$column = sanitize_key( $column );
$results = $wpdb->get_results(
    "SELECT `{$column}` FROM $wpdb->posts WHERE post_status = 'publish'"
);
```

### Real-World Example: Dynamic Column in JOIN

This is a common pattern that causes errors:

```php
// WRONG - Will cause SQL syntax errors
$dimension = 'page';
$metrics = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT t1.%s as %s, t1.clicks, t1.impressions
         FROM ( SELECT %s, SUM(clicks) as clicks FROM {$wpdb->prefix}stats 
                GROUP BY %s) as t1
         LEFT JOIN ( SELECT %s, SUM(clicks) as clicks FROM {$wpdb->prefix}stats 
                     GROUP BY %s) as t2
         ON t1.%s = t2.%s",
        $dimension, $dimension, $dimension, $dimension, 
        $dimension, $dimension, $dimension, $dimension
    )
);
// Results in invalid SQL with quoted column names and syntax errors
```

Correct approach:

```php
// RIGHT - Sanitize and use directly with backticks
$dimension = sanitize_key( $dimension );
$sql = "SELECT t1.`{$dimension}` as `{$dimension}`, t1.clicks, t1.impressions
        FROM ( SELECT `{$dimension}`, SUM(clicks) as clicks 
               FROM {$wpdb->prefix}stats GROUP BY `{$dimension}`) as t1
        LEFT JOIN ( SELECT `{$dimension}`, SUM(clicks) as clicks 
                    FROM {$wpdb->prefix}stats GROUP BY `{$dimension}`) as t2
        ON t1.`{$dimension}` = t2.`{$dimension}`";

// Only use prepare for actual data values
$metrics = $wpdb->get_results(
    $wpdb->prepare(
        $sql,
        // Add any data parameters here
    )
);
```

## Performance: Caching Database Results

While caching database results often improves performance, it's not always necessary or beneficial.

### When NOT to Cache Database Results

Caching should be skipped in these scenarios:

1. **Real-time data requirements**: When the data must always be current (e.g., live statistics, user balances)
2. **Infrequently accessed data**: When the cost of maintaining the cache outweighs the benefit of caching
3. **User-specific data**: When the data is specific to a user and not reusable across users
4. **Simple, fast queries**: When the query is already optimized and fast (e.g., primary key lookups)
5. **Highly volatile data**: When the data changes so frequently that the cache would be constantly invalidated
6. **Low-traffic features**: When the feature is rarely used and doesn't impact overall performance

### Example: When to Skip Caching

```php
/**
 * Get current user's latest notification
 * - No caching needed: User-specific and requires real-time data
 */
function get_user_latest_notification( $user_id ) {
    global $wpdb;
    
    return $wpdb->get_row(
        $wpdb->prepare(
            "SELECT * FROM {$wpdb->prefix}notifications 
             WHERE user_id = %d 
             ORDER BY created_at DESC 
             LIMIT 1",
            $user_id
        )
    ); // phpcs:ignore WordPress.DB.DirectDatabaseQuery.NoCaching -- User-specific real-time data, caching not appropriate
}
```

## PHPCS Rules and Annotations

WordPress Coding Standards include rules about database queries that may trigger PHPCS warnings.

### Direct Database Calls

PHPCS often flags direct database calls with warnings like:

```
Direct database call detected
```

This is because WordPress often has built-in functions that should be used instead of direct DB queries.

#### When to Use Annotations

If you've determined a direct DB call is necessary, use the PHPCS ignore annotation:

```php
// For a single line
$results = $wpdb->get_results( $query ); // phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery

// For a block of code
// phpcs:disable WordPress.DB.DirectDatabaseQuery.DirectQuery
$results1 = $wpdb->get_results( $query1 );
$results2 = $wpdb->get_results( $query2 );
// phpcs:enable WordPress.DB.DirectDatabaseQuery.DirectQuery
```

#### Better: Justify Your Decision

A better practice is to explain why you're bypassing the rule:

```php
$results = $wpdb->get_results( 
    $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}custom_table WHERE user_id = %d", $user_id )
); // phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery -- Custom table not accessible via WP functions
```

### Uncached Database Call

PHPCS may also flag uncached database calls:

```
Direct database call without caching detected
```

#### When to Ignore This Rule

Use the following annotation when caching isn't necessary, always including a clear justification:

```php
// phpcs:ignore WordPress.DB.DirectDatabaseQuery.NoCaching -- Data changes frequently, caching not appropriate
$current_value = $wpdb->get_var(
    $wpdb->prepare(
        "SELECT meta_value FROM $wpdb->postmeta WHERE post_id = %d AND meta_key = %s",
        $post_id,
        $meta_key
    )
);
```

Common justifications for skipping caching:
- "User-specific data not reusable across requests"
- "Real-time data required for accurate reporting"
- "Simple primary key lookup, caching overhead exceeds query cost"
- "Low-traffic admin-only feature"
- "Data changes frequently, cache would be quickly invalidated"

## Advanced Techniques

### Batching Database Queries

Batching database operations is beneficial in these scenarios:

1. **Bulk Data Processing**: When you need to insert, update, or delete multiple rows at once (e.g., importing data, bulk user operations, migration scripts)

2. **Atomic Operations**: When multiple related changes need to succeed or fail together (e.g., transferring credits between accounts, complex inventory adjustments)

3. **Performance-Critical Operations**: When reducing the number of database round-trips is essential for performance (e.g., high-traffic areas of your application)

4. **Background Processing**: When processing large datasets through cron jobs or background tasks where efficiency matters more than immediate feedback

5. **API Endpoints**: When an API call might trigger multiple database changes that should be treated as a single unit of work

### Implementation Examples

#### Basic Transaction Example

```php
global $wpdb;

// Start a transaction
$wpdb->query( 'START TRANSACTION' );

try {
    // Operation 1
    $wpdb->update(
        $wpdb->prefix . 'user_balance',
        array( 'balance' => $new_sender_balance ),
        array( 'user_id' => $sender_id ),
        array( '%f' ),
        array( '%d' )
    );
    
    // Operation 2
    $wpdb->update(
        $wpdb->prefix . 'user_balance',
        array( 'balance' => $new_receiver_balance ),
        array( 'user_id' => $receiver_id ),
        array( '%f' ),
        array( '%d' )
    );
    
    // Operation 3
    $wpdb->insert(
        $wpdb->prefix . 'transaction_log',
        array(
            'sender_id' => $sender_id,
            'receiver_id' => $receiver_id,
            'amount' => $amount,
            'timestamp' => current_time( 'mysql' ),
        ),
        array( '%d', '%d', '%f', '%s' )
    );
    
    // Commit if all operations succeeded
    $wpdb->query( 'COMMIT' );
    return true;
} catch ( Exception $e ) {
    // Rollback if any operation failed
    $wpdb->query( 'ROLLBACK' );
    error_log( 'Transaction failed: ' . $e->getMessage() );
    return false;
}
```

#### Bulk Insert with Prepared Statements

For extremely large datasets, a single multi-value INSERT can be more efficient:

```php
function batch_insert_records( $records ) {
    global $wpdb;
    $table = $wpdb->prefix . 'my_table';
    
    // Abort if no records
    if ( empty( $records ) ) {
        return false;
    }
    
    // Start building the query
    $query = "INSERT INTO $table (column1, column2, column3) VALUES ";
    $placeholders = array();
    $values = array();
    
    // Build placeholders and flatten values
    foreach ( $records as $record ) {
        $placeholders[] = "(%s, %d, %f)";
        $values[] = $record['column1'];
        $values[] = $record['column2'];
        $values[] = $record['column3'];
    }
    
    // Combine the query
    $query .= implode( ', ', $placeholders );
    
    // Execute with all values at once
    return $wpdb->query(
        $wpdb->prepare( $query, $values )
    );
}
```

### Performance Considerations

- **Memory Usage**: For very large datasets, process in chunks to avoid memory exhaustion
- **Lock Contention**: Long-running transactions can cause lock contention; keep transactions as short as possible
- **Error Handling**: Always include robust error handling and logging
- **Deadlock Prevention**: Structure operations consistently to prevent deadlocks (e.g., always update tables in the same order)

### Using Database Metadata

For queries on WordPress native objects such as posts, use WordPress functions when appropriate:

```php
$posts = get_posts( array(
    'post_type' => 'product',
    'meta_query' => array(
        array(
            'key' => 'color',
            'value' => 'blue',
            'compare' => '=',
        ),
    ),
) );
```