# Overview

This document describes the API that is used to access and update data stored in the custom database schema.

# API Namespace

The API namespace is:

```
/nc-data/v1
```

The intention here is that:

* The prefix `nc` is used for our database tables and in other locations within the code
* `data` implies that there may be other types of APIs offered by the North Commerce plugin, this API is intended for data access
* `v1` implies that this is version 1 of the API.

Note: we have a version that corresponds to the database schema. This is intentionally independent from the API version.

# Database Schema Mapping

The objects accessed and updated via the API are intended to map into our custom database schema. Generally, database table and column names are used consistently, and if you know one, you know the other.

## DB vs API Naming Conventions

The database and APIs have different requirements and conventions about their naming, so a set of rules is provided to map one to the other. These rules include:

* Database tables are prefixed with the following string: `{$wpdb->prefix}_nc_`
* Database names use _'s as a separator, API names use -.

The following are examples of how these rules look in practice:

| DB Table Name | API Entity Name |
| ------------- | --------------- |
| wp_nc_products | products |
| wp_nc_product_types | product-types |
| wp_nc_bump_details | bump-details |

## DB Table References and API Entities

With a database table definition, it's common to see references to other tables. For example:

```
'table' => 'products',
'cols' => [
	'sku' => ['type' => 'VARCHAR(128)'],
	'product_type_id' => [
		'type' => 'REF',
		'ref_table' => 'product_types'
	],
	'name' => ['type' => 'VARCHAR(255)'],
```

In this case, the `products` table references the `product_types` table via the column `product_type_id`.

When retreiving a `product` object via the API, the caller has two options:

a. Leave the `product_type_id` untouched and access this attribute using as if it were an opque ID.

b. Expand `product_type_id` so that it is replaced with the corresponding `product_type` object.

Consider these two examples for accessing a product with the ID 948.

```
# Option (a)
GET /nc-data/v1/products/948
{ data: {
   id: 948,
   sku: 't-shirt',
   product_type_id: 3,
   ...
  },
  ... }

# Option (b)
GET /nc-data/v1/products/948?expand=product_type
{ data: {
   id: 948,
   sku: 't-shirt',
   product_type: {
     id: 3,
     name: "Subscription",
     slug: "subscription"
   }
   ...
  },
  ... }
```

The default behavior is (a) to ensure that response payloads are as
small and quickly generated as possible. Behavior (b) is available
to reduce the need to make multiple trips to the server to return
linked objects.

# Response Format

All API responses have the following format:

```
{
  "data": <data>,
  "code": <code>,
  "meta": {
    "timing": 0.0013,
    "current_page": 1,
    "prev_page_link": "",
    "next_page_link": ""
  }
}

```

The caller should consult `code` to ensure that the response
did not contain an error.

If the status indicates that no error occured, `<data>` represents the
response from the API.

`meta` can be used to access additional details about the repsonse,
such as additional data elements. 

# GET Requests

GET requests are used to retrieve data from the custom database schema.

## Request URIs

The following request URIs are supported. 


### `/nc-data/v1/<entity name>`

Example: `/nc-data/v1/product-types`

Returns a list of all `product-types` in the system. 

### `/nc-data/v1/<entity name>/<id>`

Example: `/nc-data/v1/products/3456`

Returns a single `product` with the ID `3456`.

### `/nc-data/v1/<entity name>/<id>/<entity name>`

Example: `/nc-data/v1/orders/395/line-items`

Returns a list of `line-items` associated with the  `order` which has
the ID `395`.

There are two types of references within the database: one-to-one and
one-to-many. While querying both types of objects uses the above
format, there are some slight variations. Consider these examples:

| Relationship Type | Path                                   | Response                                                                                               |
|-------------------|----------------------------------------|--------------------------------------------------------------------------------------------------------|
| One-To-Many       | /nc-data/v1/orders/293/line-items      | List of line item objects associated with orders.                                                      |
| One-To-One        | /nc-data/v1/orders/294/shipping-detail | The single shipping-detail record from the `shipping_details` table that is associated with the order. |

When querying one-to-many relationship, both entities correspond to
the name of the table. When queryine a one-to-one relationship, the
first entity name is a parent name, the second is the ID column minus
the `_id` suffix.

For example: to query the one bump-detail record associated with the `products`
table, you would note the name of the column on the `products` table
that refers to `bump_details` is named `pubmp_detail_id`. You would
drop the `_id` and replace `_` with `-`.

The result is:

```
/nc-data/v1/products/948/bump-detail
```


## Query Parameters

The following query parameters are available to optimize the response
for the caller.

### `expand=<entity name>,...`

Use to instruct the API to turn referenced ID values into the
object being referenced.

You can nest entity expansions by using a dot notation. 

Example:

```
/nc-data/v1/orders?expand=payment-status,order-changes.order-change-type'
```

The above will expand both `payment-status` and `order-changes` on `orders`. It will also expand `order-change-type` on
`order-changes`.

### `filter=<column>:<op>:<value>,...`


Used to filter the response of request that return mulitple values. For
example, to return product variants that are both visible and have a
price greater than $80.00, the caller would use the following `filter`
query parameter:

```
filter=visible:eq:1,visible:gt:80
```

See: [[Data-Api-Filtering]]

The following operators are supported:


| filter operator | SQL Operator |
|-----------------|--------------|
| eq              | =            |
| equal           | =            |
| lt              | <            |
| gt              | >            |
| like            | LIKE         |


### `sort=<column>:<desc|asc>,...`

Used to order the response when a series of objects are queries. For
example, to order products by their create date and SKU, the caller
would set the `sort` query parameter to:

```
sort=created:desc,sku:asc
```

### `limit=<n>`

Limits the maximum number of objects that can be returned from a
single call.

### `page=<n>`

Requests page `<n>` of the results. To access the first page of the
results set `page=1`.

# POST Requests

POST Requests are used to create and update North Commerce database
records. All POST requests expect a JSON encoded object within the
body of the POST. The data provided in this object describes the
changes that will occur to the remote data records.

## POST Body Definition

JSON encoded objects consist of a series of fields that meet the
following criteria:

* Simple attribute values
* A nested object that has the same name as an `id` based attribute, with the
  `_id` suffix removed.
* An array of objects that has an attribute name that matches an
  existing entity. The entity must have a reference to the object being created
  or updated.

While these rules sound complex in prose, in practice they are
straightforward. Consider this POST body from the `orders` entity:

```
{
  shipping: 45.99,
  shipping_detail: {
    carrier_name: "FedEx",
    tracking_number: "2d6aa16ee5e01392ff4c39c24b4be091",
    estimated_delivery: "2022-05-03 13:00:00"
  },
  line_items: [
    { amount: 19.95, quantity: 3,
      description: "PHP Rocks T-shirt",
      product_wp_post_id: 9384 },
    { amount: 19.95, quantity: 1,
      description: "JavaScript is way cooler than PHP T-shirt",
      product_wp_post_id: 4948 },
  ]
}
```

`shipping` is an example of a simple attribute. `shipping_detail` is
an example of an object that would normally be referenced via
`shipping_detail_id`. Finally `line_items` refers to the `line-items`
entity.

## Controlling Update vs. Create Semantics

Reords that contain an `id` column are interpeted as calls to update a
record and will fail if an entity with that `id` does not
exist. Records that do not include an `id` column are interpeted as
requests to make a new object with the specified attributes.

The `id` for the top level object must be included in the URI. For
nested objects, the `id` value is included in the object definition
itself.

It's possible to deliver a POST body that contains both update and
create requests. Consider this POST Body:

```
POST /nc-data/v1/orders/9484
{ shipping: 19.95,
  order_changes: [
    { value: 19.95, description: "Shipping cost set",
      order_change_type_id: 2 }
  ]
}
```

In this example, the URI specifies that `order` 9484 should have its
`shipping` attribute updated to $19.95. It also contains an array of
`order_changes` that will be processed. The `order_change` object
provided does not contain an `id` value, therefore, it is interepted
as request to create a new `order_change` record.

## Request URIs

### `/nc-data/v1/<entity name>`

Create a new `<entity name>`.

Example:

```
POST /nc-data/v1/bump-details
{
  price: 9.99,
  quantity: 3,
  first_payment: 1.00,
  number_of_payments: 4,
  payment_requency_id: 3
}
```

This request creates a new `bump-details` record.


### `/nc-data/v1/<entity name>/<id>`

Updates the `<entity name>` identified by `<id>`

Example:

```
POST /nc-data/v1/products/3884
{
  bump_product_id: 383
}
```

Sets the `bump_product_id` on the `product` with the `id` value 3884.

The above request assumes that a `bump_product` with the `id` 383
exists. Rather than issuing a `POST` request to create the
`bump_product` followed by an additional request to update the
`bump_product_id` to the created value, it's possible two complete
both these operations in a single request:


```
POST /nc/data/v1/products/3884
{
  bump_product: {
    price: 9.99,
    quantity: 3,
    first_payment: 1.00,
    number_of_payments: 4,
    payment_requency_id: 3
  }
}
```

## Query Parameters

### `expand=<entity name>,...`

By default, the modified object is returned in the response. You can use
the `expand` query parameter in the same way as described in the [GET
request](#expandentity-name) to include additional detail in the
response.

It's also possible to pass `expand=none` to have only the ID
of the object modified returned in the response. This is helpful if
the caller doesn't need the updated payload record, and wishes to
ensures responses are as compact as possible.

Examples:

Create a new order record and return the details of the updated
record:

```
POST /nc-data/v1/orders
...
```

Create a new order record and return the details of the order and its
line items:

```
POST /nc-data/v1/orders?expand=line-items
...
```

Create a new order record and return the ID of the order record an no
other data.

```
POST /nc-data/v1/orders?expand=none
...
```
