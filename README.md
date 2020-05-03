# Getting started with FQL (Fauna Query Language)

These are my notes on Fauna DB and FQL. Part explanation of basic things, part cookbook with snippets that I don't want to forget. I'm learning as I go along. Don't hesitate to correct me.

To get started with Fauna just open a free account and go to the [dashboard](https://dashboard.fauna.com/).

If you get stuck sign into the Fauna Slack. There's people from all timezones hanging there and willing to help. You can also ask in StackOverflow. Hopefully Fauna will create an indexable forum some day.

Sections:
* <a href="#using-the-dashboard-shell">Using the dashboard shell</a>
* <a href="#naming-conventions">Naming conventions</a>
* <a href="#naming-conventions">Documents and collections</a>
* <a href="#about-relationships-and-references">Relationships and references</a>
* <a href="#indexes">Indexes</a>

## Using the dashboard shell
You can input FQL queries directly in the web dashboard using a shell. It's a great way of figuring out how FQL works and it remembers the queries you've made.

### Shortcuts:
* Submit current query scope: <kbd>CMD</kbd> + <kbd>Enter</kbd>
* Previous query: <kbd>CMD</kbd> + <kbd>UP</kbd>
* Next query: <kbd>CMD</kbd> + <kbd>DOWN</kbd>
* Undo: <kbd>CMD</kbd> + <kbd>Z</kbd>

In Windows just change <kbd>CMD</kbd> for <kbd>Ctrl</kbd>

## Naming conventions
AFAIK there are no naming conventions in Fauna but this is what I've been doing and it has stuck:
* Collections go in PascalCase. Eg: `MyFavoriteThings`.
* Document properties go in camelCase. Eg: `musicalInstrument`.
* References in a document always end with `Ref` or `Refs` in the case of arrays. Eg: `userRef` or `productsRefs`.
* Indexes go in snake_case with collections and properties as they exist in the database. Eg: `JazzArtists_by_musicalIntrument`.

## Documents and collections
Fauna doesn't have tables like traditional relational databases. The smallest unit of data is a schemaless document. These are organized in collections which are essentially buckets of documents with no order whatsoever. Fauna databases can also have children databases.

This is what a document looks like:
```js
{
  "ref": Ref(Collection("Fruits"), "264471980339626516"),
  "ts": 1588478985090000,
  "data": {
    "name": "Mango"
  }
}
```

Each document is composed by:
* A reference (`ref` in short). More on refs later, but this represents a location in the database. In this case  `Ref(Collection("Fruits"), "264471980339626516")` represents the document with the id `264471980339626516` in the `Fruits` collection.
* A timestamp `ts` in nanoseconds.
* Some `data` that looks and behaves pretty much like JavaScript objects with some special Fauna types like `Ref`.

[More about documents here.](https://docs.fauna.com/fauna/current/api/fql/documents)

Let's create a collection with [CreateCollection](https://docs.fauna.com/fauna/current/api/fql/functions/createcollection):
```js
CreateCollection({ name: "Fruits" })

// Result:

{
  "ref": Collection("Fruits"),
  "ts": 1588478932770000,
  "history_days": 30,
  "name": "Fruits"
}
```

As you can see collections are similar to documents with some special properties. Let's keep the defaults for now but you can [read more about collections here](https://docs.fauna.com/fauna/current/api/fql/collections).

Now let's create a document with [Create](https://docs.fauna.com/fauna/current/api/fql/functions/create):
```js
Create(
  Collection("Fruits"),
  {
    data: {
      name: "Mango"
    }
  }
)

// Result:

{
  "ref": Ref(Collection("Fruits"), "264471980339626516"),
  "ts": 1588478985090000,
  "data": {
    "name": "Mango"
  }
}
```
* `Create` is the function used to create documents
* `Collection('Fruits')` gets a reference to the collection `Fruits`
* `{data: {name: 'Mango'}}` is the document you want to create

### Create a document with a predefined id
First you need to retrieve a new unique id in the Fauna cluster with [NewId](https://docs.fauna.com/fauna/current/api/fql/functions/newid):
```js
NewId()

// Result:

"264517098691101203"
```
And then you can create your new document with a reference to the provided `id` instead of the reference to the `Fruits` collection:
```js
Create(
  Ref(Collection("Fruits"), "264517098691101203"),
  {
    data: {
      name: "Apple"
    }
  }
)
```

## Relationships and references
Relationships in Fauna are modelled with references. A [Ref](https://docs.fauna.com/fauna/current/api/fql/functions/ref) is a type of data in Fauna which points to a location in the database. For documents this reference includes an `id` property which would be the `264517098691101203` in the previous example.

Remember that a reference is not a thing, it's a pointer to a thing (documents, collections, etc).

If you want to retrieve a document from a reference you need to use the [Get](https://docs.fauna.com/fauna/current/api/fql/functions/get) function:
```js
Get(
  Ref(
    Collection("Fruits"),
    "264471980339626516"
  )
)

// Result:

{
  "ref": Ref(Collection("Fruits"), "264471980339626516"),
  "ts": 1588478985090000,
  "data": {
    "name": "Mango"
  }
}
```
So this is how you could model a relationship between documents:
```js
Create(
  Collection("People"),
  {
    data: {
      name: "Pier",
      favoriteFruitRef: Ref(Collection("Fruits"), "264471980339626516")
    }
  }
)
```
Or:
```js
Create(
  Collection("People"),
  {
    data: {
      name: "Pier",
      favoriteFruitsRefs: [
        Ref(Collection("Fruits"), "264471980339626516"),
        Ref(Collection("Fruits"), "467478980389696500")
      ]
    }
  }
)
```
**Note:** If you're coming from SQL you might be tempted to store raw ids in your documents but it's much better to use the native `Ref` type as it will make your FQL life simpler.

## Indexes
Retrieving one document at a time with `Get` won't get you very far. Since collections are just buckets of documents, you need to use indexes to create some order, retreive multiple documents, sort, filter, and more.

The combination of a collection and an index is probably the closest you will get to a table of a relational database. The big difference being that rows can only belong to a single table but Fauna documents can belong to as many indexes as you need.

Let's create our first index with [CreateIndex](https://docs.fauna.com/fauna/current/api/fql/functions/createindex):
```js
CreateIndex({
  name: "all_Fruits",
  source: Collection("Fruits")
})

// Result:

{
  "ref": Index("all_Fruits"),
  "ts": 1588482162446000,
  "active": true,
  "serialized": true,
  "name": "all_Fruits",
  "source": Collection("Fruits"),
  "partitions": 8
}
```

As you can see indexes are also a special type of document. Pretty much everything in Fauna is a document.

After adding some more fruits to your `Fruits` collection let's retreive all the references using the `all_Fruits` index we just created:
```js
Paginate(
  Match(
    Index("all_Fruits")
  )
)

// Result:

{
  "data": [
    Ref(Collection("Fruits"), "264471980339626516"),
    Ref(Collection("Fruits"), "264476914201133588"),
    Ref(Collection("Fruits"), "264476925331767828"),
    Ref(Collection("Fruits"), "264476941393854996"),
    Ref(Collection("Fruits"), "264476977509958164"),
    Ref(Collection("Fruits"), "264476994575532564"),
    Ref(Collection("Fruits"), "264477007396471316")
  ]
}
```
Woah woah woah there's a lot going on there.

Let's break it down. I've found sometimes it makes more sense to go backwards with FQL:
* [Index](https://docs.fauna.com/fauna/current/api/fql/functions/index) returns a reference to an index.
* [Match](https://docs.fauna.com/fauna/current/api/fql/functions/match) match is used to "execute" the index and returns a [Set](https://docs.fauna.com/fauna/current/api/fql/sets). You can use it to pass parameters the the index for filtering.
* [Paginate](https://docs.fauna.com/fauna/current/api/fql/functions/paginate) takes a `Set` and returns a page or list of references.

Later on we'll see how to get actual documents instead of references and how to use `Match` for filtering.

### Page size
By default `Paginate` returns pages with 64 items. You can define how many items you want to receive with `size`:
```js
Paginate(
  Match(Index('all_Fruits')),
  {
    size: 3
  }
)

// Result:

{
  "after": [
    Ref(Collection("Fruits"), "264476941393854996")
  ],
  "data": [
    Ref(Collection("Fruits"), "264471980339626516"),
    Ref(Collection("Fruits"), "264476914201133588"),
    Ref(Collection("Fruits"), "264476925331767828")
  ]
}
```
In this case there are more results than fit in a single page so Fauna gives us the next document to use as a cursor. See the [Paginate](https://docs.fauna.com/fauna/current/api/fql/functions/paginate) docs for more on using cursors.

### Unique values
You can also use indexes to determine that a property of a document must be unique in the whole collection much like the `UNIQUE` constraint in SQL. This must be defined when creating the index:
```js
CreateIndex({
  name: "Users_by_email",
  source: Collection("Users"),
  terms: [
    {field: ["data", "email"]}
  ],
  unique: true,
})
```
Even if you never use that index to retrieve documents it will ensure the `email` property in the `data` part of your document is unique accross the `Users` collection. Fauna will return an error if you try to insert a document that breaks that constraint.

This `["data", "email"]` represents a path. We'll see more about these later but you can think of those like `data/email` or even `data.email`.

## Gettings multiple documents from references
Ok so we know how to get a list of references from an index. How do we actually get documents?

If you've used a functional language you might have come across `map` which executes a function on each item of an array and returns a new array. For example in JavaScript we use [`myArray.map()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map).

Fauna also has a [Map](https://docs.fauna.com/fauna/current/api/fql/functions/index) function and we can use it to retrieve an array of documents from an array of references:
```js
Map(
  Paginate(Match(Index('all_Fruits'))),
  Lambda("fruitRef", Get(Var("fruitRef")))
)

// Result:

{
  "data": [
    {
      "ref": Ref(Collection("Fruits"), "264471980339626516"),
      "ts": 1588478985090000,
      "data": {
        "name": "Mango"
      }
    },
    {
      "ref": Ref(Collection("Fruits"), "264476914201133588"),
      "ts": 1588483690410000,
      "data": {
        "name": "Apple"
      }
    },
    // etc
  ]
}
```
Don't panic, let's break this down.

We already understand from the previous example that `Paginate(Match(Index('all_Fruits')))` returns an array of references, right? So `Map` iterates over this array, executes `Lambda` on each item, and returns a new array.

This part is a bit tricky: `Lambda("fruitRef", Get(Var("fruitRef")))`:
* [Lambda](https://docs.fauna.com/fauna/current/api/fql/functions/lambda) is used in FQL to define anonymous functions.
* `Lambda("fruitRef"...` defines a function parameter. It could be named anything: `fruit`, `X`, `Tarzan`, etc. The name is irrelevant. In this case the parameter will receive a single document reference that `Map` will pass to `Lambda` from `Paginate`.
* `Var("fruitRef")` evaluates the variable named `fruitRef` in the context of the function. You couldn't simply use `fruitRef` or `"fruitRef"` because FQL wouldn't know what do with it. With [Var](https://docs.fauna.com/fauna/current/api/fql/functions/lambda) you explicitly tell Fauna to find a variable in the current context.
* Finally `Get` receives the document reference from `Var` and returns a document. This document is returned by `Lambda` to `Map` to form an array of documents.

So consider this JavaScript example:
```js
const newArray = myArray.map((item) => doSomething(item));
// which is equivalent to:
const newArray = myArray.map(function (item) {
  return doSomething(item);
});
```
If you were using the JavaScript Fauna driver you could write something like this instead of using `Lambda`:
```js
const result = await client.query(
	q.Map(
	  q.Paginate(q.Match(q.Index('all_Fruits'))),
	  (fruitRef) => q.Get(fruitRef)
	)
)
```