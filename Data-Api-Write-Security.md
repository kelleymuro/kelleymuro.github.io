# Overview

Here are some considerations for locking down write access to our api

1. Need to support converting create requests to lookups

   * Public user always creates a `customer`, internally this is mapped to lookup or create

2. Rejecting requests to change certain fields

   * line_items.amount can't be changed ever
   * line_items.quantity can only be changed when the order is in a certain state
   
3. Auto calculate fields on the server

   * product_variants.quantity and product.quantity is updated when an order is marked as unfulfilled
   * order.total, order.subtotal is updated when line_items.quantity or line_items.product_variant_id is changed
   * line_items.amount is updated when line_items.quantity is changed
   * order.order_status and order.payment_status are set to sane values when an order is initially created

# Specific Write Access Rules

##  customers

state: new

  * public: MUST provide e-mail and marketing_optin
  * system: looks up e-mail and converts 'new request' to lookup if email exists

state: existing

 * public: can update marketing_optin


##  customer_payment_methods

state: new

 * public: MUST provide customer_id  and payment_provider_id, can provide:
    * token
    * last4
    * brand
    * expiration_date
    * expiration_year
    
state: existing

  * public:  can updated:
    * token
    * last4
    * brand
    * expiration_date
    * expiration_year

##  addresses

state: new

  * public: MUST provide customer_id AND  can provide:
      * is_default_shipping
      * is_default_billing
      * first_name
      * last_name
      * phone
      * addressline1
      * addressline2
      * city
      * state
      * zipcode
      * country_id
      
state: existing

  * public: can update:
      * is_default_shipping
      * is_default_billing
      * first_name
      * last_name
      * phone
      * addressline1
      * addressline2
      * city
      * state
      * zipcode
      * country_id    

##  shipping_details

state: new

  * public/customer: must provide rate_type_id, can provide:
    * ezpost_carrier_id
    * ezpost_shipping_id
    * fixed_rate_id
    
  * system:
    * verifies that combination of rate_type_id and other fields provided make sense
    * calculates handle_fee, shipping_fee
    
state: existing

  * public/customer: can update:
    * rate_type_id
    * ezpost_carrier_id
    * ezpost_shipping_id
    * fixed_rate_id
    
  * system:
    * verifies that combination of rate_type_id and other fields provided make sense
    * calculates handle_fee, shipping_fee


##  orders

state: new

  * public/customer: MUST provide customer_id, can provide:
      * customer_payment_method_id
      * billing_address_id
      * shipping_address_id
      
  * system: will fill in sane defaults for the remaining fields

state: Created

  * public/customer: can update
      * customer_payment_method_id
      * billing_address_id
      * shipping_address_id


state: <all other order_status_id's>

  * public/customer: can NOT make changes to the order


##  order_transactions

state: new

 * public/customer: MUST provide order_id. can provide:
     * identity_token
   System fills in sane values for:
     * order_transaction_status_id
     * amount

state: Building

  * public: can update:
      * identity_token
      * order_transaction_status_id (system verfies state changes with external provider)

state: <all remaining>

  * public: no changes allowed


##  line_items

state: new & order.order_status = Created & order.order_transaction.order_transaction_status = Building

  * public/customer: MUST provide order_id, product_variant_id, quantity
    * System:
      * generates sane values for line_items.amount, line_items.description
      * updates: orders.total, order.subtotal,
      * updates: order_transaction.total
      
state: existing & order.order_status = Created & order.order_transaction.order_transaction_status = Building

  * public/customer:
    Can update:
      * quantity
      * product_variant_id
    System:
      * updates line_item.amount, line_items.description
      * updates order.total, orders.subtotal,
      * updates: order_transaction.total
    
state: <all remaining>

  * public/customer: no changes allowed


##  customer_tags

state: <all>

  * public/customer: no changes allowed

##  customer_changes

state: <all>

  * public/customer: no changes allowed


##  order_changes

state: <all>

  * public/customer: no changes allowed
