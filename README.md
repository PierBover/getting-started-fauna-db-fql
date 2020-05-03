# Getting started with FQL (Fauna Query Language)

These are my notes on Fauna DB and FQL. Part explanation of basic things, part cookbook with snippets that I don't want to forget. I'm learning as I go along. Don't hesitate to correct me.

To get started with Fauna just open a free account and go to the [dashboard](https://dashboard.fauna.com/).

If you get stuck I recommend you to sign into the Fauna Slack. There's people from all timezones hanging there and willing to help. A chat is not ideal though as a lot of useful information gets lost. You can also ask in StackOverflow but hopefully Fauna will create an indexable forum some day.

## Using the dashboard shell
You can input FQL queries directly in the dashboard using a shell. It remembers the queries you've made and it's a great way of figuring out how FQL works.

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
* References in a document always end with `Ref`. Eg: `userRef`.
* Indexes go in snake_case with collections and properties as they exist in the database. Eg: `JazzArtists_by_musicalIntrument`.

## Documents and Collections
Fauna doesn't have tables like in traditional relational databases. The smallest unit of data is a schemaless document. These are organized in collections which are essentially folders of documents with no order whatsoever. Fauna databases can also have children databases.

This is what a document looks like:
```
{
  "ref": Ref(Collection("Fruits"), "264471980339626516"),
  "ts": 1588478985090000,
  "data": {
    "name": "Mango"
  }
}
```

Each document is composed by:
* A reference (`ref` in short). More on refs later, but this essentially represents a location in the database. In this case this represents the document with the id `264471980339626516` in the `Fruits` collection.
* A timestamp `ts` in nanoseconds.
* Some `data` that looks and behaves pretty much like JavaScript objects with some special Fauna types like `Ref`.

[More about documents here.](https://docs.fauna.com/fauna/current/api/fql/documents)

Let's create a collection with [CreateCollection](https://docs.fauna.com/fauna/current/api/fql/functions/createcollection):
```js
CreateCollection({ name: "Fruits" })
// result:
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
	// this gets a reference to the collection "Fruits"
	Collection('Fruits'),
	// your new document
	{
		data: {
			name: 'Mango'
		}
	}
)
// result:
{
  "ref": Ref(Collection("Fruits"), "264471980339626516"),
  "ts": 1588478985090000,
  "data": {
    "name": "Mango"
  }
}
```

## About relationships and references
Relationships in Fauna are modelled with references. A [Ref](https://docs.fauna.com/fauna/current/api/fql/functions/ref) is a type of data in Fauna which points to a location in the database. For documents this reference includes an `id` property which would be the `264471980339626516` in the previous example.

You can get a reference to a collection using [Collection](https://docs.fauna.com/fauna/current/api/fql/functions/collection) which you can use to create documents as you have seen in the previous example.

Remember that a reference is not a thing, it's a pointer to a thing. If you want to retrieve a document from a reference you need to use the [Get](https://docs.fauna.com/fauna/current/api/fql/functions/get) function:
```js
Get(Ref(Collection("Fruits"), "264471980339626516"))
// result:
{
  "ref": Ref(Collection("Fruits"), "264471980339626516"),
  "ts": 1588478985090000,
  "data": {
    "name": "Mango"
  }
}
```
So now we can start modeling relationships:
```js
Create(
	Collection('People'),
	{
		data: {
			name: 'Pier',
			favoriteFruitRef: Ref(Collection('Fruits'), "264471980339626516")
		}
	}
)
```
Or one to many:
```js
Create(
	Collection('People'),
	{
		data: {
			name: 'Pier',
			favoriteFruitsRefs: [
				Ref(Collection('Fruits'), "264471980339626516"),
				Ref(Collection('Fruits'), "467478980389696500")
			]
		}
	}
)
```
**Note:** If you're coming from SQL you might be tempted to store raw ids in your documents but it's much better to use the native `Ref` type as it will make your FQL life simpler.

## Indexes
Getting one document at a time with `Get` gets old pretty fast. Since collections are just buckets of documents with no order you need to use indexes to retreive multiple documents, sort, filter, and more.

Let's create our first index with [CreateIndex](https://docs.fauna.com/fauna/current/api/fql/functions/createindex):
```js
CreateIndex({
  name: "all_Fruits",
  source: Collection("Fruits")
})
// result:
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

Add some more fruits to your database and then let's retreive all the references using the `all_Fruits` index:
```js
Paginate(
	Match(
		Index('all_Fruits')
	)
)
// result:
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
Woah woah woah there's a lot going on there. Let's break it down. I've found sometimes it makes more sense to go backwards with FQL:
* [Index](https://docs.fauna.com/fauna/current/api/fql/functions/index) returns a reference to an index
* [Match](https://docs.fauna.com/fauna/current/api/fql/functions/match) match is used to "execute" the index and returns a [Set](https://docs.fauna.com/fauna/current/api/fql/sets). Later on we'll see how it's used to pass parameters for filtering.
* [Paginate](https://docs.fauna.com/fauna/current/api/fql/functions/paginate) takes a `Set` and, in this case, returns a page of references.

Later on we'll see how to get actual documents and not references.

### Page size
By default `Paginate` returns pages with 64 documents. You can define how many documents you want to receive with `size`:
```js
Paginate(
	Match(
		Index('all_Fruits')
	),
	{
		size: 3
	}
)
//result:
{
  // since there are more fruits than fit in a page Fauna gives us the next document to use as a cursor
  "after": [
    Ref(Collection("Fruits"), "264476941393854996")
  ],
  "data": [
    Ref(Collection("Fruits"), "264471980339626516"),
    Ref(Collection("Fruits"), "264476914201133588"),
    Ref(Collection("Fruits"), "264476925331767828")
  ]
}
````
### Unique values
You can also use indexes to determine that a property of a document must be unique in the whole collection. This must be defined when creating the index like this:
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

These are the basics about indexes. You can read [more about indexes here](https://docs.fauna.com/fauna/current/api/fql/indexes) and follow [this official tutorial](https://docs.fauna.com/fauna/current/tutorials/indexes/) to learn to sort and filter.

## Gettings multiple documents from references
Ok so we know how to get a list of references with an index. But how do we actually get the documents?

If you've used a functional language you might have come across `map` which executes a function on each item of an array and returns a new array. In JavaScript for example this is done with [`myArray.map()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map).

Fauna also has a [Map](https://docs.fauna.com/fauna/current/api/fql/functions/index) function which we'll use to do exactly that.
```js
Map(
  Paginate(Match(Index('all_Fruits'))),
  Lambda("fruitRef", Get(Var("fruitRef")))
)
// results:
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
    {
      "ref": Ref(Collection("Fruits"), "264476925331767828"),
      "ts": 1588483701000000,
      "data": {
        "name": "Banana"
      }
    },
    // etc
  ]
}
```
Let's break this down:
* We already understand from the previous example that `Paginate(Match(Index('all_Fruits')))` returns an array of references, right?
* `Map` will iterate over the array returned by `Paginate`, execute the function defined by `Lambda` on each item, and return a new array formed by the return values of `Lambda`.

### Understanding `Lambda`
This part is a bit tricky: `Lambda("fruitRef", Get(Var("fruitRef")))`.
* `Lambda("fruitRef"...` defines a new variable in the context of the body of the anonymous function. It could be named anything `fruit`, `X`, `Tarzan`, etc. The name is irrelevant.
* `Var("fruitRef")` evaluates the variable named `fruitRef` in the context of the function. We know it's a reference to a document because it comes from the `Paginate` function.
* Finally `Get` receives the document reference from `Var` and returns a document. This document in turn is returned by `Lambda` to `Map` to form a new array of documents.

Maybe it will make more sense with this JavaScript example:
```js
const newArray = myArray.map((item) => doSomething(item));
// which is equivalent to:
const newArray = myArray.map(function (item) {
	return doSomething(item)
});
````
