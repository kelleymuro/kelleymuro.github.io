# Overview

This page describes how a user can interactively run pre-configured
example code.  To run an example, `ssh` to your WordPress server and
change to the `north-commerce` directory. From there, you can issue
commands with the following shape:

```
ssh <my WordPress instance>
$ cd /var/www/html/wp-content/plugins/north-commerce
$ php -f tools/run_example.php <path to example file> <name>=<value> ...
```

See below for a series of transcripts that show these examples can be executed.

# Load Test Data

To load test data, access your WordPress server. If this is a remote server, then this is probably done via ssh. If your running a local server, then you can probably access your server via a terminal of some sort.

Change in the plugin source code directory and run the following command:

```
ssh <my linux server>
$ cd /var/www/html/wp-content/plugins/north-commerce
$ php -f tools/populate_test_data.php num_users=3 num_products=4 num_coupons=3
...
```

The populate_test_data.php command takes some time. It will create the number of users requested, and 10 test orders for each of these users.

# Sample Transcripts

## Query Adresses

```
# See what addresses and countries are currently in the system
php -f tools/run_example.php includes/db/examples/All_Entities.php table=addresses expand= limit=1

# Same as before, but expand the details about country associate with each address
php -f tools/run_example.php includes/db/examples/All_Entities.php table=addresses expand=country limit=2

# Pull out addresses that are in Mexico
php -f tools/run_example.php includes/db/examples/Addresses_By_Country.php country_abbreviation=MX limit=1
```

## Query Customers

```
# Preview all customers
php -f tools/run_example.php includes/db/examples/All_Entities.php table=customers expand limit=3

# See all customers that have a name (first, last or e-mail) of Linda
php -f tools/run_example.php includes/db/examples/Customers_By_Name.php name=Linda limit=1

# See all customers that have an e-mail that looks like a test e-mail
php -f tools/run_example.php includes/db/examples/Customers_By_Name.php name='@testing.northplugins.com' limit=1

# All customers who are named Linda and have an address in Mexico
php -f tools/run_example.php includes/db/examples/Customers_By_Name_And_Country.php name=Linda country_abbreviation=MX
```

## Query Products

```
# See all products with a minimum of $50 of profit. Sort by profit, descending.
php -f tools/run_example.php includes/db/examples/Products_With_Profit.php min_profit=50

# See all products with a Size option
php -f tools/run_example.php includes/db/examples/Products_With_Options.php option_name=Size option_value=*

# See all products with a Size=Large option
php -f tools/run_example.php includes/db/examples/Products_With_Options.php option_name=Size option_value=Large
```

