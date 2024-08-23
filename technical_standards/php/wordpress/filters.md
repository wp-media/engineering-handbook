---
notion_page: https://www.notion.so/wpmedia/WordPress-Filters-Best-practices-179498fcf4434006baae8d9b9950a5a7?pvs=4
title: WordPress Filters - Best practices
---

# WordPress Filters: Best practices 

Filters are one of the two types of Hooks provided by WordPress as a way for third-parties to modify data dynamically during execution of the code. Read [the official documentation](https://developer.wordpress.org/plugins/hooks/filters/) for more details.

Filters are a pwoerful feature of WordPress that we extensively use within our plugins to make them customizable easily.

## PHP Types and filters
### The issue with filters loose typing

WordPress filters have no typing check in place. This means that it’s not possible to trust the returned value of a filter, because anyone could have modified it using the filter and returned an unexpected type. 

In consequence, this can cause errors with the code using this value, especially if it expects a specific type, for example a method with typed parameters. 

Our plugins are used by thousands of users, on millions of websites so we must expect that someone will mess with our filters at some point. Moreover, the error will usually be reported from a file within our plugin's codebase, even if the mistyped value is introduced by another third-party.

### Safeguarding returned value
To prevent this issue, whenever we implement a custom filter in our code, we need to safeguard the returned filter value before using it further: in case the filtered value is not as expected, we should fallback to the default one. 

The safeguarding mechanism is usually based on types, but can be stricter depending on the context: for instance, a price must be a float but also be stricly positive.

Our current approach to this is to use the [apply-filters-typed library](https://github.com/wp-media/apply-filters-typed). It has the benefit of adding logs in debug mode to warn developers of filters being misused. Complex value checks can be handled with [the custom type validation feature](https://github.com/wp-media/apply-filters-typed?tab=readme-ov-file#create-your-own-type-validation) of the library.

Note that the same approach applies to callbacks re-using the argument being filtered.

#### Previous practice
Before the introduction of this library, or when complex validation is needed, we were explicitely validating the returned value as follows:
```php
// Variable with the default value for the filter
$default = 10;

$value = apply_filters( 'rocket_filter', $default );

// Check type of the returned value
if ( ! is_int( $value ) ) {
	// Returned value is not the expected type, set it back to default
	$value = $default;
}
```
In addition to checking for type, we could also test against the value, for example in the case where we expect it to be a positive integer only. All those cases should now be handled with [apply-filters-typed library](https://github.com/wp-media/apply-filters-typed). When reworked, code using the old approach can be rewritten to leverage our new standard.

### Automated tests of filters
To validate the safeguarding of the filtered value, it is recommended adding a serie of fixtures while writing unit/integration tests for the method using it.

The fixtures can pass different values to the filter and validate against the expected output. The following list of values can be used:

- `""` (empty string)
- `null` (null)
- `[]` (empty array)
- `{}` (empty object)
- `"text"` (string instead of number)
- `[1, 2, 3]` (array instead of single value)
- `-1` (negative number)
- `9999999999` (very large number)
- `NaN` (Not a Number)
- `Infinity` (Infinity)
- `true` (boolean true)
- `"!@#$%^&*()"` (string with special characters)

For unit test, you can filter the value in the following way, thanks to BrainMonkey library:

```php
Filters\expectApplied( 'rocket_filter' )->andReturn( $fixture_value );
```

For integration test, we can directly use the WordPress functions:
```php
add_filter( 'rocket_filter', [ $this, set_value() ] );

public function set_value() {
	return $fixture_value;
}
```

