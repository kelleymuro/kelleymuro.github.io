# Basic Entity Access Usage

This page covers how to get started with EntityAccess inside of the North Commerce plugin. 

EntityAccess is what we call our system for to create, update or delete any data in the North Commerce tables.

Within EntityAccess lives our [[Read Security Rules|Data-Api-Read-Security]], our [[Write Security Rules|Data-Api-Write-Security]] and [[Filtering|Data-Api-Filtering]]. 

Currently this gives us everything in one place, consistency and inheritance automatically as we add new tables, columns and integrations. The goal is to provide an easy to use "API" that in the future will provide a universal standard for all future North Commerce add-ons from us or 3rd party developers.

# Getting Started

In order to access entity access you first need to create an instance of `North_Commerce_Db_Agent`. Here is a typical implementation for getting the required functions and classes. 

```
$agent = North_Commerce_Db_Agent::instance();
$ea = $agent->entityAccess();
```
Just like that you now have access to all the database tables and can easily `expand` on any other data from any other table via the foreign keys.

## Types of access

Inside of EntityAccess you get a few different types of of access. 
The code for this can be found in `/includes/db/class-north-commerce-db-entity-access.php`

1. `get`  $ea->get($table, $criteria, $options = []);
2. `ref`  $ea->ref($id, $options = []);
3. `list` $ea->list($table, $criteria, $options = []);
4. `listAll` $ea->listAll($table, $criteria, $options = []);
5. `count` $ea->count($table, $criteria);
6. `create` $ea->create($table, $values, $options = []);
7. `update` $ea->update($table, $existing, $changes, $options = []);
8. `store` $ea->store($table, $changes, $options = []);
9. `delete` $ea->delete($table, $criteria, $options = []);

The arguments for these functions are as described:

`$table` - This is the table you are wanting to access. For a list of tables you can go to the North Commerce plugin `settings -> API Docs` or you can look at the `Schema.php` file inside of the North Commerce plugin.

`$criteria` - This is where you could filter the data you want. Let's say you want to list all of the products that match a certain category and product status. That would look something like this.

```
$filter = [ 'product_categories.category.name' => 'Men's Shoes', 'product_status_id' => ProductStatuses::published()->id ];
```
The filter above looks inside of the product_categories table and we use a dot notation to access the name of the category. We also look at the product status id to make sure it matches the id of our published product status from it's own table in the database. This gives us all Men's Shoes that are published so we know we won't get any products that have a `Draft` or `Scheduled` status

`$options` - This is an array typically used to expand the respoonse object or array to show more relevant data for a specific query. This could be getting shipping details from an order and/or customer information from a specific order. Examples below.

### Get

Let's say for example you want to get a single order from the database. You can use `get` to achieve that. 

In this example I will get a specific order with a certain filters. The filter in this case will be the id of the order. 

```
$agent = North_Commerce_Db_Agent::instance();
$ea = $agent->entityAccess();
$ea->get('orders', ['id' => 'or199v7s3fjb2cmbpq0d4pxrxm00'], []);
```
Above we are access the `id` from the orders table with the value of `or199v7s3fjb2cmbpq0d4pxrxm00` and providing no options. This returns the core columns from the orders table for the order that matches the id specified. 

`RESPONSE`
```
array(
    "id" => "or199v7s3fjb2cmbpq0d4pxrxm00",
    "order_number" => "1030",
    "customer_id" => "cu4wqnp8fg1ya00ck6jbd88kk928",
    "shipping_address_id" => "addj2xaw79cex00ekjzsdh4kcxq5",
    "billing_address_id" => "ad27dh20mxq9y6cnnmgz1gzkznd2",
    "order_status_id" => "1",
    "payment_status_id" => "5",
    "customer_payment_method_id" => NULL,
    "shipping" => "80.82",
    "tax" => "0.00",
    "subtotal" => "1876.35",
    "total" => "1957.17",
    "paid" => "0.00",
    "created" => "2024-01-02 21:27:58"
);
```

You will notice we get all the columns from the orders table. If this is all we need then the response is small, fast and we can do whatever we want with it. 

In another case we may want to get the customer that this order belongs to. This is where the `options` array comes in. Inside of the `options` array lives `expand`. With expands we can easily add any data from any table that the orders table references, like `customers`

### Get with `Expand`

I'm going to add to my existing $ea->get() code. 

```
$agent = North_Commerce_Db_Agent::instance();
$ea = $agent->entityAccess();
$options = ['expand' => 'customer']; 
`expand` is required and then I can reference any table or even a column from a table
$ea->get('orders', ['id' => 'or199v7s3fjb2cmbpq0d4pxrxm00'], $options);
```
The expand array can get long and complex so we usually create a new variable for it called `$options`.

`RESPONSE`

```
array(
    "id" => "or199v7s3fjb2cmbpq0d4pxrxm00",
    "order_number" => "1030",
    "customer_id" => "cu4wqnp8fg1ya00ck6jbd88kk928",
    "shipping_address_id" => "addj2xaw79cex00ekjzsdh4kcxq5",
    "billing_address_id" => "ad27dh20mxq9y6cnnmgz1gzkznd2",
    "order_status_id" => "1",
    "payment_status_id" => "5",
    "customer_payment_method_id" => NULL,
    "shipping" => "80.82",
    "tax" => "0.00",
    "subtotal" => "1876.35",
    "total" => "1957.17",
    "paid" => "0.00",
    "created" => "2024-01-02 21:27:58",
    "customer" => array(
        "id" => "cu4wqnp8fg1ya00ck6jbd88kk928",
        "customer_number" => "1003",
        "first_name" => "Linda",
        "last_name" => "HLWGQH",
        "email" => "nc-test-HLWGQH@testing.northplugins.com",
        "phone" => "533-555-7070",
        "country_code" => "+1",
        "wp_user_id" => "6",
        "total_amount_spent" => NULL,
        "total_number_of_orders" => NULL,
        "has_active_subscription" => NULL,
        "marketing_optin" => "0",
        "created" => "2024-01-02 21:27:57"
    )
);

```
You are able to expand on as many referenced columns as you want. If there is an id in the column you can just remove the `_id` and expand that data.  That looks like this:

``` 
$options = ['expand' => 'customer,shipping_address,customer_payment_method,order_status'];
```

