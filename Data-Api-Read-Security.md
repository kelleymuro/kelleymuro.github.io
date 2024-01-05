
# Overview

This document defines the approach to limiting access to data
returned by the Data API.

The security model assumes that the API is being accessed in one of
three ways:

* public - No WordPress authentication is present
* customer - User is authenticated as a Customer of a NorthCommmerce store.
* admin - User is an Administrator of a NorthCommerce store

Here are common patterns the security framework needs to address:

1. Customer `alice` can see her own orders, but can not see `bob`'s.

2. All users can browse `product_variants` in a store, however only an admin
can view `product_variants` that are marked as invisible. No users,
including admins, can view `product_variants` that are marked as
deleted.

3. All users can view `products`, however only admins can view the
`cost_of_goods` field on this entity.

# Security Framework Prerequisites

The security framework requires the following features to function.

1. *Advanced Filtering.* Advanced Filtering is what allows the API to
   filter out data that is not visible to users.

2. *Authentication.* The framework assumes that users can authenticate
   their identity with the API. This may be accomplished via the
   standard WordPress Cookie authentication mechanism, or via a custom
   API key generated for each user in the system. The custom API key
   method is convenient for user testing, the WordPress cookie
   approach may be more straightforward to implement in front-end
   JavaScript context.

# Entity Re-organization

Currently, the database schema is implemented a nested array that has the
following shape:

```
  'line_items' => [
    'cols' => [
      'amount' => ...,
      'quantity => ...
    ]
  ],
  ...

```

To ensure that the security framework is readable, each entity should
be relocated to its own class. These may live in
`includes/db/entities`.

The class may have a structure such as this:

```
class NcEntityLineItems extends NcEntity {
  public function cols() {
   return [
    'amount' => ...,
    'quantity' => ...
   ]
  }
}
```

The schema file is then updated to have the shape:

```
  'line_items' => new NcEntityLineItems(),
  ...
```

This is functionally equivalent to the current configuration, but
makes space for implementing the required security code.

# Query Context

When a request is made to an entity, certain facts are established before
the entity is queried from the database. These include:

* Columns to retrieve from the database
* Filters to narrow down the results returned

In the security code described below, this information is bundled
together into a query context.

# Security Through Context Management

Within each entity's class there are three security related methods methods:

```
class Nc<Entity> extends NcEntity {
  public function cols() { ... }

  public function preparePublicRead($qc) { ... }

  public function prepareCustomerRead($qc, $customer) { ... }

  public function prepareAdminRead($qc, $admin) { ... }

}
```

Each of the `prepare<Role>Read` functions is given an opportunity to
adjust the `QueryContext` to control how the final query that will be
executed against the database.

Consider how we can implement the above scenarios using this strategy.

## Limit Users To Their Own `line_items`

```
public class NcEntityLineItems extends NcEntity {
  public function cols() { ... }

  public function preparePublicRead($qc) {
    throw new NcNoAccessException();
  }

  public function prepareCustomerRead($qc, $customer) {
    $filter = $qc->filter();
	$new_filter = [
	  'and',
	    ['eq', 'order.customer.wp_user_id', $customer->ID],
		$filter
	];

	$qc->setFilter($filter);
	return $qc;
  }

  public function prepareAdminRead($qc, $admin) {
    return $qc;
  }
}
```

In the above code, `preparePublicRead` throws an exception, which
disallows access to this API endpoint from non-logged in customers.

In the above code, `prepareCustomerRead` updates the filters by adding
an overriding `and` expression with the following shape:

```
  (order.customer.wp_user_id = <customer id>) AND
  (whatever filter was requested by the API caller
```

Only orders that belong to the customer will meet the first part of
this newly crafted filter clause. If a user requested additional
filtering, such as only showing `line_items` with a specific product,
then that filter clause is still honore.

If a user attempts to access another customer's order, the result will
be a filter that returns no results. For example:

```
# Customer 9934 tries to look up `line_items` for customer 8372
['and',
  ['eq', 'order.customer.wp_user_id', 9934], # [A]
  ['eq', 'order.customer.wp_user_id', 8372], # [B]
  ]
```

In the above example `[A]` is added by the security framework, and
`[B]` is added by the customer.

The result is a request to show orders that are owned by both `9934`
`AND` `8372`, which is a logical impossibility.


## Limit Access to `product_variants`

```
public class NcProductVariants extends NcEntity {
  public function cols() { ... }

  public function preparePublicRead($qc) {
    $filter = $qc->filter();
	$new_filter = [
	  'and',
	    ['eq', 'visible', true],
		['eq', 'deleted, null],
		$filter
	];

	$qc->setFilter($filter);
	return $qc;
  }

  public function prepareCustomerRead($qc, $customer) {
    return $this->preparePulicRead($qc);
  }

  public function prepareAdminRead($qc, $admin) {
    $filter = $qc->filter();
	$new_filter = [
	  'and',
		['eq', 'deleted, null],
		$filter
	];

	$qc->setFilter($filter);
	return $qc;
  }
}
```

In the above code, `preparePublicRead` adds the requirement that the
`product_variant` must be both `visible` and non-`deleted`.
`prepareCustomerRead` delegates to `preparePublicRead` because it has
the same logic.

Finally, `prepareAdminRead` adds a filter to show only
non-`deleted` entities. The `admin` UI may opt to hide invisible
`product_variants` by explicitly setting this filter. However, from a
security perpective, this isn't required.


## Remove Sensitive attributes from `products`

```
public class NcProducts extends NcEntity {
  public function cols() { ... }

  public function preparePublicRead($qc) {
    $qc->removeCol('cost_of_goods');
	return $qc;
  }

  public function prepareCustomerRead($qc, $customer) {
    return $this->preparePulicRead($qc);
  }

  public function prepareAdminRead($qc, $admin) {
	return $qc;
  }
}
```

The above code adjusts the `QueryContext` by removing a column from
the entity returned. The above code does not adjust the filters
associated with the query, so all data requested by all customers will
be returned.

In practice, the `prepare` related functions above would most likely
adjust both columns and filters associated with the `QueryContext` to
match the required security profile for `products`.
