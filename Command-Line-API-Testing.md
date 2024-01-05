# Overview

This page explains how you can test the API using command line tools.

You should have `curl` and [jq](https://stedolan.github.io/jq/) installed before running the commands below.

You may be able to install `jq` using any of the following:

```
// On various flavors of linux
$ sudo yum install jq
$ sudo apt install jq

// On Mac
$ sudo brew install jq
$ sudo port install jq
```

# Loading Test Data

You can run the examples below, but doing so before you have test data will result in far less interesting results. To load test data, access your WordPress server. If this is a remote sever, then this is probably done via `ssh`. If your running a local server, then you can probably access your server via a terminal of some sort.

Change in the plugin source code directory and run the following command:

```
ssh <my linux server>
$ cd /var/www/html/wp-content/plugins/north-commerce
$ php -f tools/populate_test_data.php num_users=3 num_products=4
...
```

The `populate_test_data.php` command takes some time. It will create the number of users requested, and 10 test orders for each of these users.

# GET API Queries

After loading test data, run this query to ensure data is loaded:

```
curl -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders?limit=2' | jq . 
{
  "code": "ok",
  "meta": {
    "timing": 0.0012,
    "next_page_link": "https://nc.i2x.us/wp-json/nc-data/v1/orders?page=2&limit=2",
    "prev_page_link": false
  },
  "data": [
    {
      "id": "2286",
      "created": "2022-03-15 12:55:40",
      "customer_wp_user_id": "7887",
      "order_status_id": "4",
      "payment_status_id": "5",
      "shipping_detail_id": "812",
      "shipping": "43.48",
      "tax": null,
      "total": "33464.70",
      "paid": null
    },
    {
      "id": "2287",
      "created": "2022-03-15 12:55:40",
      "customer_wp_user_id": "7890",
      "order_status_id": "4",
      "payment_status_id": "5",
      "shipping_detail_id": "813",
      "shipping": "99.87",
      "tax": null,
      "total": "2910.32",
      "paid": null
    }
  ]
}
```

You can use the  `next_page_link` to see the next page of results:

```
$ curl -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders?page=2&limit=2' | jq .
```

The following command will show you just the order ids of the first 10 orders:

```
$ curl -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders?limit=10' | jq -r '.data[] | .id'
2286
2287
2288
2289
2290
2291
2292
2293
2294
2295
```

We can grab one of these IDs to experiment further. I'm using the last ID in the list, `2295` in the examples below.

To find view the line items associated with this order you can execute:

```
$ curl -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders/2295/line-items' | jq .
{
  "code": "ok",
  "meta": {
    "timing": 0.0014,
    "next_page_link": false,
    "prev_page_link": false
  },
  "data": [
    {
      "id": "5280",
      "created": "2022-03-15 12:55:45",
      "order_id": "2295",
      "product_wp_post_id": "5410",
      "description": "Test Product DHHMYC: ",
      "amount": "534.18",
      "quantity": "13"
    },
    {
      "id": "5281",
      "created": "2022-03-15 12:55:45",
      "order_id": "2295",
      "product_wp_post_id": "5411",
      "description": "Test Product XJCRAU: ",
      "amount": "164.53",
      "quantity": "28"
    }
  ]
}
```

You can get the order details by querying the API endpoint `/orders/2295`:

```
$ curl -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders/2295' | jq .
{
  "code": "ok",
  "meta": {
    "timing": 0.0014
  },
  "data": {
    "id": "2295",
    "created": "2022-03-15 12:55:45",
    "customer_wp_user_id": "7914",
    "order_status_id": "4",
    "payment_status_id": "5",
    "shipping_detail_id": "821",
    "shipping": "85.83",
    "tax": null,
    "total": "11637.01",
    "paid": null
  }
}
```

You can expand this response to include the `order_status` and `order_changes` by using the `expand` query parameter:

```
$ curl -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders/2295?expand=order_status,order-changes' | jq .
{
  "code": "ok",
  "meta": {
    "timing": 0.0033
  },
  "data": {
    "id": "2295",
    "created": "2022-03-15 12:55:45",
    "customer_wp_user_id": "7914",
    "order_status_id": "4",
    "payment_status_id": "5",
    "shipping_detail_id": "821",
    "shipping": "85.83",
    "tax": null,
    "total": "11637.01",
    "paid": null,
    "order_status": {
      "id": "4",
      "created": "2022-03-02 14:06:54",
      "name": "Unfulfilled"
    },
    "order_changes": [
      {
        "id": "2455",
        "created": "2022-03-15 12:55:45",
        "order_id": "2295",
        "order_change_type_id": "1",
        "value": "65.15",
        "description": "Good news, we Order Placed"
      },
      {
        "id": "2456",
        "created": "2022-03-15 12:55:45",
        "order_id": "2295",
        "order_change_type_id": "2",
        "value": "63.77",
        "description": "Good news, we Payment Processed"
      },
      {
        "id": "2457",
        "created": "2022-03-15 12:55:45",
        "order_id": "2295",
        "order_change_type_id": "3",
        "value": "77.32",
        "description": "Good news, we Shipping Rate Calculated"
      }
    ]
  }
}
```

See the [Data API Guide](https://github.com/kelleymuro/north-commerce/wiki/Data-Api-Guide) for the full set of query parameters you can use to interact with the API.

# POST API Queries

A warning:

> :warning: The POST api currently allows unlimited access to the
> database. Anyone with API access can create an update any entity.
> This open behavior will be going away, however, the semantics of creating an
> updating content will remain the same.


POST queries are a bit trickier to test because:

1. You often need to leverage existing IDs of entities in the database

2. You have to craft JSON object to send as the data to create/upload.

With that said, let's do this!

First, let's pick an order to update. This API request finds the last order that was created in the system:

```
$ curl  -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders?limit=1&sort=created:desc' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.0012,
    "filter": {
      "1": 1
    },
    "has_param_filter": "no",
    "next_page_link": "https://nc.i2x.us/wp-json/nc-data/v1/orders?page=2&limit=1&sort=created%3Adesc",
    "prev_page_link": false
  },
  "data": [
    {
      "id": "3037",
      "created": "2022-03-17 13:03:53",
      "customer_wp_user_id": "10867",
      "order_status_id": "4",
      "payment_status_id": "5",
      "shipping_detail_id": "1561",
      "shipping": "51.88",
      "tax": null,
      "total": "13227.82",
      "paid": null
    }
  ]
}
```

Let's update the tax and total of this order:

```
cat > /tmp/body.json <<JSON
{ "tax": 2.00,
  "total": 13229.82 }
JSON
curl -H 'Content-Type: application/json' -d @/tmp/body.json -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders/3037' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.0119
  },
  "data": {
    "id": "3037",
    "created": "2022-03-17 13:03:53",
    "customer_wp_user_id": "10867",
    "order_status_id": "4",
    "payment_status_id": "5",
    "shipping_detail_id": "1561",
    "shipping": "51.88",
    "tax": "2.00",
    "total": "13229.82",
    "paid": null
  }
}
```

Let's update the tracking number of this order.  Note the use of `expand` to see the `shipping_detail` record nested
within this order.


```
cat > /tmp/body.json <<JSON
{ 
  "shipping_detail": {
    "id": 1461,
    "tracking_number": "XYZZY"
  }
}
JSON
curl -H 'Content-Type: application/json' -d @/tmp/body.json -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders/3037?expand=shipping-detail' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.0201
  },
  "data": {
    "id": "3037",
    "created": "2022-03-17 13:03:53",
    "customer_wp_user_id": "10867",
    "order_status_id": "4",
    "payment_status_id": "5",
    "shipping_detail_id": "1461",
    "shipping": "51.88",
    "tax": "2.00",
    "total": "13229.82",
    "paid": null,
    "shipping_detail": {
      "id": "1461",
      "created": "2022-03-17 12:52:31",
      "carrier_name": "USPS",
      "tracking_number": "XYZZY",
      "estimated_delivery": "2022-03-24 07:00:00"
    }
  }
}
```

Finally, let's add a new line item to this existing order. To do this, we'll need to know what possible product IDs we can pick from.
Normally, we'd query the `products` entities, however, we're currently using WP Posts for products so we have to find the relevant
`wp_post` ID another way. Let's pick one from another oldest `line_item` in our system:

```
curl  -s 'https://nc.i2x.us/wp-json/nc-data/v1/line-items?limit=1&sort=created:desc' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.0013,
    "filter": {
      "1": 1
    },
    "has_param_filter": "no",
    "next_page_link": "https://nc.i2x.us/wp-json/nc-data/v1/line-items?page=2&limit=1&sort=created%3Adesc",
    "prev_page_link": false
  },
  "data": [
    {
      "id": "7328",
      "created": "2022-03-17 13:03:53",
      "order_id": "3037",
      "product_wp_post_id": "7566",
      "description": "Test Product CIURIB: ",
      "amount": "176.72",
      "quantity": "45"
    }
  ]
}
```

Now we can use our `order` ID from above and the product `product_wp_post_id` from this existing line item to craft a new line item:

```
cat > /tmp/body.json <<JSON
{ 
  "line_items": [
    { "product_wp_post_id": 7566,
      "description": "My New Product",
      "quantity": 3,
      "amount": "9.99" }
  ]
}
JSON
curl -H 'Content-Type: application/json' -d @/tmp/body.json -s 'https://nc.i2x.us/wp-json/nc-data/v1/orders/3037?expand=line-items' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.0105
  },
  "data": {
    "id": "3037",
    "created": "2022-03-17 13:03:53",
    "customer_wp_user_id": "10867",
    "order_status_id": "4",
    "payment_status_id": "5",
    "shipping_detail_id": "1461",
    "shipping": "51.88",
    "tax": "2.00",
    "total": "13229.82",
    "paid": null,
    "line_items": [
      {
        "id": "7327",
        "created": "2022-03-17 13:03:53",
        "order_id": "3037",
        "product_wp_post_id": "7565",
        "description": "Test Product MCXTLO: ",
        "amount": "746.22",
        "quantity": "7"
      },
      {
        "id": "7328",
        "created": "2022-03-17 13:03:53",
        "order_id": "3037",
        "product_wp_post_id": "7566",
        "description": "Test Product CIURIB: ",
        "amount": "176.72",
        "quantity": "45"
      },
      {
        "id": "7329",
        "created": "2022-03-17 13:07:12",
        "order_id": "3037",
        "product_wp_post_id": "7566",
        "description": "My New Product",
        "amount": "9.99",
        "quantity": "3"
      }
    ]
  }
}
```

Our examples have used orders so far, however, we aren't limited to `order` related tables. Let's create a new tag:

```
cat > /tmp/body.json <<JSON
{ 
  "slug": "brilliant-ideas",
  "name": "Brilliant Ideas"
}
JSON
curl -H 'Content-Type: application/json' -d @/tmp/body.json -s 'https://nc.i2x.us/wp-json/nc-data/v1/tags' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.008
  },
  "data": {
    "id": "1",
    "created": "2022-03-17 13:09:48",
    "slug": "brilliant-ideas",
    "name": "Brilliant Ideas"
  }
}
```

And let's make a new `product`:

```
cat > /tmp/body.json <<JSON
{ 
  "sku": "XT-4849",
  "name": "Excellent T-Shirt",
  "description": "A most excellent T-shirt",
  "published": "2022-07-01 03:00:00",
  "bump_detail": {
    "price": 9.99,
    "quantity": 1
  }
}
JSON
curl -H 'Content-Type: application/json' -d @/tmp/body.json -s 'https://nc.i2x.us/wp-json/nc-data/v1/products' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.024
  },
  "data": {
    "id": "1",
    "created": "2022-03-17 13:12:34",
    "sku": "XT-4849",
    "product_type_id": null,
    "name": "Excellent T-Shirt",
    "description": "A most excellent T-shirt",
    "permalink": null,
    "scheduled": null,
    "published": "2022-07-01 03:00:00",
    "deleted": null,
    "status_id": null,
    "payment_detail_id": null,
    "bump_detail_id": "1",
    "customs_detail_id": null,
    "base_price": null,
    "compare_price": null,
    "cost_of_goods_price": null,
    "profit": null,
    "quantity": null,
    "weight": null,
    "is_physical_product": null
  }
}
```

Finally, let's attach the `tag` to the `product` by creating a new `product_tags` row:

```
cat > /tmp/body.json <<JSON
{ 
  "product_id": 1,
  "tag_id": 1
}
JSON
curl -H 'Content-Type: application/json' -d @/tmp/body.json -s 'https://nc.i2x.us/wp-json/nc-data/v1/product-tags' | jq .
{
  "code": "ok",
  "message": "ok",
  "meta": {
    "timing": 0.0116
  },
  "data": {
    "id": "1",
    "created": "2022-03-17 13:17:36",
    "product_id": "1",
    "tag_id": "1"
  }
}
```

