# Overview & Goals

Ensuring a secure system is at the core of the North Commerce
plugin. Much of the security policy defined by the plugin is enforced
by the `EntityAccess` object. `EntityAccess` is a utility class that
allows for relatively easy reading and writing of data into North
Commerce's custom tables. The data-based REST api directly plugs into
`EntityAccess` as does much of server side code. This allows a
significant amount of code to benefit from the security features built
into `EntityAccess`.

The security provided by `EntityAccess` is accomplished through four
specific mechanisms noted below.

A key principle of the framework's security is that new security rules
can be added to the plugin, and existing ones audited, with relatively
low effort. The goal is to be both confident about our security
posture, and have the ability to add new security rules as they are
identified with ease.

# User Perspectives

User Perspectives allows `EntityAccess` to provide different levels of
data access to different classes of users. We currently have the
following types of User Perspectives defined:

* Public: this is a user who has not been authenticated in any
  way yet.

* Customer: this is a user who has authenticated the North Commerce
  and has been identified as a customer of the web property that is
  using the North Commerce Plugin.
  
* Administrator: this is an authenticated user who has administrative rights
  on the server.
  

See: [Security: User
Perspectives](https://github.com/kelleymuro/north-commerce/wiki/Security:-User-Perspectives)
for an example of how User Perspectives can function.

## Status

**Fully Implemented**

User Perspectives are implemented in the plugin.  Every interaction
with `EntityAccess` establishes a User Perspective and code can check
the current perspective to provide differing behavior.

# pubid Usage

Consider this snippet of the JavaScript code that runs on the
`/checkout` page:

```
+---[public/js/lib/checkout-manager.js:35]
| return api.get('customers', {
|                 filter: "email:eq:" + customerInfo.email
|             }).then(found => {
|                 if(found.length == 0) {
|                     return api.post('customers', {
|                         email: customerInfo.email,
|                         marketing_optin: customerInfo.marketing_optin,
|                     });
|                 } else {
|                     return found[0];
|                 }
+---
```

This code looks up a customer record from the data REST api via the
`api.get` function. The customer entity that is returned has a unique
`id` value.

Typically, this `id` value would be a sequential number, for
example: 3938. Providing a user back this number creates a variety of
issues:

1. The user can use the number to guess and inspect other user
   IDs. For example, the user may run an `api.get` on 3937 or 3939 to
   learn about those user identifiers.
   
2. An attacker may learn some information about a customer just by
   looking at the user's identifier. For example, if a user identifier
   is 202, that may imply the store has relatively few customers in
   its lifetime.
   

One way to address the above issues is to make use of non-sequential,
non-numeric identifier. For example, if the customer id returned was:

cueh8qg1846yw007f37jz723ebk7

an attacker would not be able to guess possibly valid IDs of another
customer, no would they learn any metadata about what that identifier
represents.

The `cu` prefix could be helpful for debugging, as every table could
make use of its own prefix and we could easily recognize that an ID
that began with `or` is an order ID and not a customer ID.

We refer to this format of id as a 'pubid' as this identifier is
generally safe to share with the public.

Other terms for this id would include GUID or UUID.

## Implementation Details

Pubid values are implemented using the following strategy:

1. The core of the id are 16 random bytes returned by PHP's
  [random_bytes](https://www.php.net/random_bytes) function. This
  function is flagged as being suitable for use by cryptographic
  functions, which means that it's a valid source of secure data.
2. The 16 bytes generate 128 bits of randomness, which is the standard
   proposed by [RFC4122](https://www.ietf.org/rfc/rfc4122.txt) and
   used in common UUID implementations.
3. The random byte are encoded in base 32. UUIDs are typically encoded
   in base 16 (hex), however base 32 requires fewer characters to store
   in the database. The choice of base 16 or 32, as with other
   formatting conventions, has no impact on how secure the IDs are.
4. The system uses [Crockford encoding](https://www.crockford.com/base32.html)
   to replace potentially ambiguous letters 'o', 'l', 'i' and 'u' with
   'w', 'x', 'y' and 'z'.
5. The base 32 encoded string is padded to 26 characters with
   additional random digits as needed.
5. The id is prefixed with a two letter code that is unique for each
   table in the system. For example, all records in the `customers`
   table have a prefix of `cu`. This allows for basic validation of
   IDs. For example, a pubid that starts with `or` can never be a
   customer ID.

This strategy results in a globally unique, secure, 28 character
string that is made up of a table prefix and unambiguous characters.
  

## Status

**Fully Implemented**

# API Read Security 

Consider the following snippet of code:

```
$results = $entityAccess->list('orders');
```

Depending on the user's perspective, we would want the following
behavior:

* Public: throw an exception. Orders can only be queried if a specific
  `pubid` is provided.
  
* Customer: all orders that the customer has purchased should be
  returned

* Administrator: All orders in the system should be returned.a

`EntityAccess` read API security is what accomplishes this goal. There
are at least two aspects of read security.

## Schema Hiding

First, parts of database schema, including columns and entire tables,
can be hidden from a User Perspective. For example, public users can
retrieve a customer record to access information needed for
checkout. However, only the `id`, `email` and `marketing_opt` in flag
are visible to this user. As non-authenticated users, that's the only
information they've provided so that's all they can use.

Another example of hidden data is the `cost_of_goods` field available
on the `products` table. Only users with the Administrator Perspective
can see these columns.

## Auto Filtering

The second core feature of API Read Security is use of
automatic-filters. These are additional database filtering criteria
that are added due to the user Perspective in question.

For example, on the `orders` table, the `CustomerPerspective` will
automatically add a filter that requires all `order` records be 
associated with the customer's id.

These automatic-filters always narrow the content returned such that
it's not possible for `EntityAccess` to leak privileged information.


## Status

This functionality is partially implemented.

Both the schema rewriting feature and support for advanced filtering
are implemented. The automatic filters are not yet implemented.

# API Write Security

Allowing different User Perspectives a secure way of modifying and
deleting data in the database is the goal of API write security.
The specific details of how API write security will be implemented are
still being defined.

There are two core phases of the Write security.

## Incoming Change Verification

The first phase consists of validating and possibly modifying incoming
write requests. For example, a request to modify the total of an order
record by a public user needs to be universally rejected. A request,
however, to allow a public user to change the line item quantity on an
order needs to be handled with more care. If the order is still being
built, then this request is valid. If the order has already
been paid for, then this request needs to be rejected.

The verification of incoming requests may require accessing external
resources. For example, a customer may claim that a Stripe order transaction
has been successfully charged. Before this status update can be
recorded in the database, we need to consult Stripe to verify this
claim.

The verification of incoming requests may involve changing that
incoming request so that a modified version of the request is
made. For example, during the checkout process public users may always
request the creation of a new customer record. The incoming request
verification framework, will catch this 'create' request and check to
see if an existing user with the provided e-mail exists. If so, then
this user is returned. 


## Database Automations

The second phase of Write Security involves implementing a series of
database automations. In the example above, after an order transaction
has been marked as successful, it's necessary to mark the order record
as having the paid status, updating the paid column and changing the
order status to `Unfulfilled`. These automations provide a mechanism
for a User Perspective who can't typically make changes to a specific
part of the schema to securely do so.

Consider another example: a public user is not allowed to modify an
order record's total. However, when this user updates the quantity on
a line item, a side effect is that the order record's total needs to
be updated as well. This happens through database automations.

## Status

Some preliminary database automations have been implemented for the
`/checkout` procedure. The verification and adjustment of incoming
write requests needs to be implemented, and the database automation
framework needs to be further formalized so it's easier to manage.





