# Overview

This page covers how filtering works in the [[Data API|Data-Api-Guide]].

When accessing data via the API, there are three types of
relationships that are exposed. They are described below.

| Relationship Type | Sample Attribute          | Sample Value                                                                                                   |
|-------------------|---------------------------|----------------------------------------------------------------------------------------------------------------|
| Simple Attribute  | `orders.total`            | Total value of the order                                                                                       |
| One-To-One        | `orders.shipping_address` | A single `address` entity or null if one isn't set.                                                            |
| One-To-Many       | `orders.line_items`       | An array of `line_item` entities. If there are no `line_items` associated with this order, the array is empty. |

# Simple Filtering

Simple filtering allows the API caller to filter results by attributes that are combined with the `and` logical operator.

The format is:

```
<attribute>:<operator>:<value>,...
```

For example, you could pull up all the variants that are visible and
have a price over $80 with the following filter:

```
visible:eq:1,price:gt:80
```

The following operators are supported:

* `eq` - equals
* `equal` - equals
* `lt` - less than
* `gt` - greater than
* `like` - SQL LIKE operator (using `%` as the wildcard)

Simple filtering is easy to compose but has significant
limitations. Which brings us to Advanced Filtering.

# Advanced Filtering

Advanced filtering allows for sophisticated queries that are
not possible with Simple Filtering. An Advanced Filter is formatted as
a JSON encoded array with the following shape:

```
[ <operator>, <argument>, ... ]
```

Each operator has its own requirements for what arguments are
expected. This array notation has the benefit being nestable, which
means that arbitrarily complex expressions can be built using it.

Here are the oeprators that are supported, as well as the definition
of their arguments:

| Operator | Arguments            | Notes                                           |
|----------|----------------------|-------------------------------------------------|
| `and`    | `expr`, ...          | Can nest any number of expressions              |
| `or`     | `expr`, ...          | Same as `and`, but with logical `or` semantics. |
| `equals` | `attribute`, `value` |                                                 |
| `lt`     | `attribute`, `value` | Less than.                                      |
| `gt`     | `attribute`, `value` | Greater than.                                   |
| `like`   | `attribute`, `value` | MySQL LIKE operator                             |
| `is`     | `attribute`, `value` | MySQL IS operator.                              |
| `not`    | `expr`               |                                                 |
| `true`   |                      | Takes no arguments, always returns `true`       |
| `false   |                      | See `true`                                      |
| `null`   | `attribute`          | True if the attribute is NULL                   |


# Filtering Formats


Filtering requests are formatted using one of three different methods:

* String Based Expressions. The expression is encoded as a
  string. This can be used for simple expressions only. 

* Associative Array Expressions. Here, the filter is encoded as an
  associative array. Like string based expressions, only simple
  filters can be expressed using this method. However, they tend to be
  cleanest format to read in PHP.
  
* Sequential Array Expressions. These expressions make use of a prefix
  notation that allow for arbitrarily complex expressions.
  
It's possible to embed Associative Array and String Based Expressions
within a Sequential Array Expression.

## String Based Expressions

String based expressions have the format:

```
<attribute>:<operator>:<value>,...
```

As described above, each clause is separted by a `,`, which is turned
into an `AND` operation.

## Associative Array Based Expressions

Associative array based expressions have the format:

```
// A
['<attribute1>' => ['<operator>', '<value>'],
 '<attribute2>' => ['<operator>', '<value>'],
 ...]
 
// B
['<attribute1>' => '<value>',
 '<attribute2>' => '<value>',
 ...]
```

In the above examples, each key value pair mapped is joined together
with an `AND` operation. If no `<operator>` is provided, `=` is
assumed.

## Sequential Array Based Expressions

The sequential array based expressions are most flexible, allowing the
user to make use of the `and` and `or` operators and supporting any
number of nested expressions.

```
[ <operator>, <argument>, ... ]
```

## Formatting Examples

```
// sku = 'ABC123'
$criteria = "sku:eq:ABC123";
$criteria = [
	'sku' => 'ABC123'
];
$criteria = ['=', 'sku', 'ABC123'];
```

```
// sku LIKE 'ABC%'
$criteria = "sku:like:ABC%";
$criteria = [
	'sku' => ['LIKE', 'ABC123']
];
$criteria = ['like', 'sku', 'ABC%'];
```

```
// sku LIKE 'ABC%' AND base_price > 100
$criteria = "sku:like:ABC%,base_price:gt:100";
$criteria = [
	'sku' => ['LIKE', 'ABC123'],
	'base_price' => ['>', 100]
];
$criteria = ['and',
			 ['like', 'sku', 'ABC%'],
			 ['>', 'base_price', 100]];
```

```
// (sku LIKE 'ABC%') OR (sku LIKE 'XYZ%')
// String notation: can't represent
// Assoc Array notation: can't represent
$criteria = ['or',
			 ['like', 'sku', 'ABC%'],
			 ['like', 'sku', 'XYZ%']];
$criteria = ['or',
			 ['sku'=> ['like', 'ABC%']],
			 ['sku'=> ['like', 'XYZ%']]];
```

```
// created > '2022-01-02' AND (sku LIKE 'ABC%' OR sku LIKE 'XYZ%')
// String notation: you can't.
// Asoc Array notation: you can't
$criteria = ['and',
			 '>', 'created', '2022-01-02',
			 ['or',
			  ['like', 'sku', 'ABC%'],
			  ['like', 'sku', 'XYZ%']]];
```


```
// line_items: created after 2022-01-02 AND contains a product tagged with 'xyz' AND belongs to customer with ID 1039
// String notation: you can't.
// Asoc Array notation: you can't
$criteria = [ 'and',
			  ['>', 'created', '2022-01-02'],
			  ['=', 'product_variant.product.product_tag.slug', 'xyz'],
			  ['=', 'order.customer.id', 1039]];
```



# Examples
Consider these examples:

```
# Search customers for name or e-mail that contains the text 'Bob'
['or', ['like', 'first_name', '%Bob%'],
       ['like', 'last_name', '%Bob%'],
       ['like', 'email', '%Bob%']]

# Search orders that belong to 'Bob'
['or', ['like', 'customer.first_name', '%Bob%'],
       ['like', 'customer.last_name', '%Bob%'],
       ['like', 'customer.email', '%Bob%']]

# Find all orders that were placed in the UK and
# have more than $10 of tax
['and',
  ['equal', 'shipping_address.country.abbreviation', 'UK'],
  ['gt', 'tax', 10]]


# Find all non-deleted products that are tagged with the `samsung`
# tag and have a profit greater than $100
['and',
  ['equal', 'product_tags.tag.slug', 'samsung']
  ['is', 'deleted', null],
  ['gt', 'profit', 100]]

# Find all product variants that are visible
# and either have more than 10 in stock or cost more than $10
['and',
 ['equal', 'visible', 1],
 ['or',
   ['gt', 'price', 10],
   ['gt', 'quantity', 10]]]

# Find all bump products that are visible and have
# quantity in stock
['and',
  ['equal', 'product_variant_type.slug', 'bump'],
  ['gt', 'quantity', 0],
  ['equal', 'visible', 1]]
```

# REST API Filtering

The REST API is currently limited to [Simple Filtering](#simple-filtering) expressions only.
The filtering expressions are passed as a string to the `filter` URL parameter. Consider these examples:

```

# Filter: products.sku = XRBRSK-001'
$ curl -s -G 'https://nc.i2x.us/wp-json/nc-data/v1/products' -d filter='sku:eq:XRBRSK-001' | jq .data
[
  {
    "created_by_wp_user_id": "2",
    "sku": "XRBRSK-001",
    "product_type_id": "1",
    "name": "Mega Book XRBRSK  (with options)",
    "description": "This is your chance to own a genuine, one of a kind Mega Book XRBRSK.",
    "slug": "xrbrsk",
    "scheduled": null,
    "published": "2022-09-16 19:07:47",
    "deleted": null,
    "product_status_id": "1",
    "has_customs_details": "0",
    "customs_detail_id": null,
    "payment_detail_id": "1",
    "base_price": "55.05",
    "compare_price": "14.31",
    "cost_margin": null,
    "quantity": null,
    "weight": "91",
    "is_physical_product": "1",
    "has_product_variants": "1",
    "id": "1",
    "created": "2022-09-16 18:59:45"
  }
]

# Filter: product.sku starts with 'DW'
$ curl -s -G 'https://nc.i2x.us/wp-json/nc-data/v1/products' -d filter='sku:like:DW%' | jq .data
[
  {
    "created_by_wp_user_id": "230",
    "sku": "DWENLG-001",
    "product_type_id": "1",
    "name": "Micro Pod DWENLG (with options)",
    "description": "This is your chance to own a genuine, one of a kind Micro Pod DWENLG.",
    "slug": "dwenlg",
    "scheduled": null,
    "published": "2022-09-28 00:06:00",
    "deleted": null,
    "product_status_id": "2",
    "has_customs_details": "0",
    "customs_detail_id": "1",
    "payment_detail_id": "25",
    "base_price": "16.26",
    "compare_price": "86.94",
    "cost_margin": "23.68",
    "quantity": "55",
    "weight": "80",
    "is_physical_product": "1",
    "has_product_variants": "1",
    "id": "29",
    "created": "2022-09-20 00:06:48"
  }
]

# Filter: product has the 'shoe' tag
$ curl -s -G 'https://nc.i2x.us/wp-json/nc-data/v1/products' -d filter='product_tags.tag.slug:eq:shoe'   | jq .data
[
  {
    "created_by_wp_user_id": "229",
    "sku": "LURVRV-001",
    "product_type_id": "1",
    "name": "Nano Book LURVRV",
    "description": "This is your chance to own a genuine, one of a kind Nano Book LURVRV.",
    "slug": "lurvrv",
    "scheduled": "2022-09-21 01:59:12",
    "published": "2022-09-24 00:06:00",
    "deleted": null,
    "product_status_id": "4",
    "has_customs_details": "0",
    "customs_detail_id": "1",
    "payment_detail_id": "24",
    "base_price": "11.13",
    "compare_price": "77.37",
    "cost_margin": "-574.66",
    "quantity": "2",
    "weight": "87",
    "is_physical_product": "1",
    "has_product_variants": "0",
    "id": "28",
    "created": "2022-09-20 00:06:48"
  }
]
```

# Notes on Relationships

Simple and One-to-One attribute relationships are generally
straightforward to work with. The dotted notation tends to work as
you'd expect. For example:

* `orders`: `total`
* `customers`: `default_billing_address.country.abbreviation`

This behavior matches the `expand` operator.

Working with One-To-Many relationships may have unexpected
consequences. Consider this example:

```
# Table: `customer`
['equal', 'addresses.country.abbreviation', 'MX']
```

`customers` have multiple addresses. For this condition to match, at
least one of these addresses must have a country abbreviation that
equals `MX`.  It's possible that the customer may have other address
records that are other than `MX`.

# See Also

See [[Command Line Example Testing]] for interactive examples of
filtering.

