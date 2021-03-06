json-path (alpha) [![Build Status](https://travis-ci.org/flitbit/json-path.png)](http://travis-ci.org/flitbit/json-path)
=========

JSON-Path utility (XPath for JSON) for nodejs and modern browsers.

You may be looking for the prior work [found here](http://goessner.net/articles/JsonPath/). This implementation is a new JSON-Path syntax building on [JSON Pointer (RFC 6901)](http://tools.ietf.org/html/rfc6901) in order to ensure that any valid JSON pointer is also valid JSON-Path.

**Warning:** This is a work in progress - I am actively adding selection expressions and have yet to optimize, but as I use it in a few other projects I went ahead and made it available via `npm`.

## Installation

[node.js](http://nodejs.org)
```bash
$ npm install json-path
```

## Basics

JSON-Path takes a specially formatted *path* statement and applies it to an object graph in order to *select* results. The results are returned as an array of data that matches the path.

Most paths start out looking like a JSON Pointer...

```javascript
// From: http://goessner.net/articles/JsonPath/
var data = {
  store: {
    book: [
      { category: "reference",
        author: "Nigel Rees",
        title: "Sayings of the Century",
        price: 8.95
      },
      { category: "fiction",
        author: "Evelyn Waugh",
        title: "Sword of Honour",
        price: 12.99
      },
      { category: "fiction",
        author: "Herman Melville",
        title: "Moby Dick",
        isbn: "0-553-21311-3",
        price: 8.99
      },
      { category: "fiction",
        author: "J. R. R. Tolkien",
        title: "The Lord of the Rings",
        isbn: "0-395-19395-8",
        price: 22.99
      }
    ],
    bicycle: {
      color: "red",
      price: 19.95
    }
  }
};
```

The pointer `/store/book/0` refers to the first book in the array of books (the one by Nigen Rees).

**Differentiator**

The thing that makes JSON-Path different from JSON-Pointer is that you can do more than reference a single point in a structure. Instead, you are able to `select` many pieces of data out of a structure, such as:

`/store/book[*]/price`
```javascript
[8.95, 12.99, 8.99, 22.99]
```

In the preceding example, the path `/store/book[*]/price` has three distinct statements:

Statement | Meaning
--- | ---
`/store/book` | Get the book property from store. This is similar to `data.store.book` in javascript.
`*` | Select any element (or property).
`/price` | Select the price property.

Starting with the original data, each statement refines the data, usually by selecting parts. As each statement is processed, it is given the results from the previous statement and may make further selections, until the final selections are returned to the caller. It works something like map-reduce; or if you like, something like list-comprehensions.

**Distinguishing Statements**

Statements are distinguished from one another using the square-brackets `[` and `]`. In many cases, the parser can infer where one statement ends and another begins, such as in the preceding example `/store/book[*]/price`. However, the parser understands the equivelant, fully specified path `[/store/book][*][/price]`.

Paths can have as many distinct statements as you need to select just the right data. Since it extends JSON-Pointer, you must take care when your path contains square-brackets as part of property names such as the following contrived example:

```javascript
var data = {
	'my[': {
		contrived: {
			'example]': { should: "mess with", your: "noodel" } } }
};
```

In this data, the property names `my[` and `example]` are valid but would cuase ambiguities for either the parser or the processing of statements. In these cases, you must use the URI fragment identifier representation described in [RFC 6901 Section 6](http://tools.ietf.org/html/rfc6901). For instance, to access `data['my['].contrived['example]'].your` you would need the path `#/my%5B/contrived/example%5D/your`.

### More Power

JSON-Path becomes more powerful with a few additional types of statements:

Statement | Meaning
--- | ---
`..` | Makes an exhaustive descent, executing the next statement against each branch of the object-graph.
`@` | Uses the user-supplied function to select or filter data.

Consider the following examples using the same preceding data:

Path | Result
--- | ---
`/store[..]/price` | Selects all prices, from books and the bicycle.
`../isbn` | Selects all ISBN numbers, wherever they are in the structure.
`/store/book[*][@]` | Selects all books, providing each to the user-supplied selection method.

**User Supplied Selection Methods**

Rather than introduce an expression syntax, JSON-Path supports the use of user-supplied selections. Mindful of the preceding data, consider the following code:

```javascript
var jpath = require('json-path')
, expect  = require('expect.js')
, data    = require('./example-data')

var p = jpath.create("#/store/book[*][@]");

var res = p.resolve(data, function(obj, accum) {
  if (typeof obj.price === 'number' && obj.price < 10)
    accum.push(obj);
  return accum;
});

// Expect the result to have the two books priced under $10...
expect(res).to.contain(data["store"]["book"][0]);
expect(res).to.contain(data["store"]["book"][2]);
expect(res).to.have.length(2);

```

The example above illustrates that a user-defined selection given to `resolve` is used by JSON-Path in place of the `@`.

