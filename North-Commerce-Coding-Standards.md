# Coding Standards & Patterns

This document describes the coding standards we will follow for building North Commerce. We will follow some WordPress coding standards and follow some of our own based on how we have implemented things specific to our project. 

# PHP Standards

# PHP Tags
```
<?php echo ... ?> 
```

# Namespaces
North Commerce namespaces will not follow WordPress convention but our own `Camel Case` 
```
namespace NorthCommerce\..\..
```
&

```
class CamelCaseNamingPattern extends CamelCase
```

# Arrays
Use bracketed arrays
```
[1, 2, 3]

```

# Brackets (Functions, Class Definitions)
We want to star the opening brace on the same line as the function or class definition
```
brackets_are_awesome {

}
```

# Templating (WIP)
North Commerce currently has and will have LOTS of different templates. I want to adopt the same mental model for templating that is used in the atomic design pattern

![Atom design pattern](https://miro.medium.com/max/720/1*yf0J0_I9tY6gUvY2KKMtjQ.png)

`Atoms` - which is all the individual markup
<br>
`Molecules` -  which is a collection of atoms
<br>
`Organisms` - which is a collection of molecules that will end up giving us the template we have on the page
<br>
`Templates` - which is the entire template finished that is then added to the page
<br>
`Pages` or `file` - that has all of the templates inside of it.

It will vary based on what feature we are working but I think it would be best to think about how to break up a page into templates by looking at what a `Molecule` is made up of and its functionality. Often we may template a smaller bit of markup or functionality. But there may be cases where we have larger components (`Organisms`) that we can break up. 

The templating engine itself is inspired by the _underscore theme and follow this pattern

```
<?php echo $this->template('file-name', 'folder/exists', [data] ) ?>
```

Each template should look like regular HTML and we will drop into php when we have to. E.g:
```
  <div class="variant__generated">
      <h3>Modify created variants.</h3>
      <div class="generated__labels result__labels">
      <div class="label result-col-3"><?php echo esc_html( 'Variant', 'north-commerce' ); ?></div>
      <div class="label result-col-2"><?php echo esc_html( 'Price', 'north-commerce' ); ?></div>
      <div class="label result-col-2"><?php echo esc_html( 'SKU', 'north-commerce' ); ?></div>
      <div class="label result-col-2"><?php echo esc_html( 'Inventory', 'north-commerce' ); ?></div>
   </div>
```
# CSS Standards
We currently are using `SASS` and `gulp` to compile that `SASS`. 

We will always style classes and never nested tags in classes. 
<br>
✅ ```.defined_class { color: black }```
<br>
❌ ```.defined_class p { color: black }```

`IDs`: Always add IDs to each element so styling can easily be overwritten

`Discussion`: How do we want to standarize class names and IDs?

# NEEDS DISCUSSION

1. Less Ajax & more vanilla javascript? 

2. Generalized Javascript event listeners: We will have lots of templates that may do the same thing so we want to generalize the Javascript so we dont have to rewrite or duplicate Javascript code that is already working. For example: If I create a shortcode that lists an array of products and I want to add one of those products, I currently can't use the add to cart functionality that has been implemented on the `Product Overview` page. 

3. Compartmentilzed or multi file Javascript - I don't want the Javascript for an entire page to be in one javascript file. I want to seperate each file to be manageable and easy to understand. If something is cart related put it in a cart.js file, if something is for an automation, add it to the automation folder in its own file, if something is for variants, put it in a variants.js file. 

4. Moving packages to composer: Easypost, Stripe?, Mollie, Twilio 

# Models
TBD