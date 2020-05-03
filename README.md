# Getting started with FQL (Fauna Query Language)

This is not an exhaustive documentation but a series of ideas and examples that can help you approach the official docs more easily. I found FQL to be difficult to grasp until it *clicks*. Each section builds on the previous ones so if you are just skimming and don't understand something you probably need to go back.

I'm also learning FQL as I go along so don't hesitate to correct me.

To get started with Fauna just open a free account and go to the [dashboard](https://dashboard.fauna.com/).

If you get stuck sign into [Fauna's community Slack](https://community-invite.fauna.com/). There's people there from all timezones willing to help. You can also ask in StackOverflow using the [`faunadb`](https://stackoverflow.com/questions/tagged/faunadb) tag.

## Sections:
* <a href="#using-the-dashboard-shell">Using the dashboard shell</a>
* <a href="#naming-conventions">Naming conventions</a>
* <a href="#naming-conventions">Documents and collections</a>
* <a href="#about-relationships-and-references">Relationships and references</a>
* <a href="#indexes">Indexes</a>
* <a href="#gettings-multiple-documents-from-references">Gettings multiple documents from references</a>
* <a href="#sorting-results">Sorting results</a>
* <a href="#filtering-results">Filtering results</a>
* <a href="#about-paths-and-using-select">About paths and using Select</a>
* <a href="#getting-custom-results-instead-of-full-documents">Getting custom results instead of full documents</a>

## Using the dashboard shell
You can input FQL queries directly in the web dashboard using a shell. It's a great way of figuring out how FQL works and it remembers the queries you've made.

#### Shortcuts:
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
* A timestamp `ts` in microseconds.
* Some `data` that looks and behaves pretty much like JavaScript objects with [some special Fauna types](https://docs.fauna.com/fauna/current/api/fql/types) like `Ref`.

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
* `Create` is the function used to create documents.
* `Collection('Fruits')` gets a reference to the collection `Fruits`.
* `{data: {name: 'Mango'}}` is the document you want to create.

See the [CRUD tutorial](https://docs.fauna.com/fauna/current/tutorials/crud) for more info on how manipulate documents.

#### Create a document with a predefined id
First you need to retrieve a new unique id in the Fauna cluster with [NewId](https://docs.fauna.com/fauna/current/api/fql/functions/newid):
```js
NewId()

// Result:

"264517098691101203"
```
And then you can create your new document with a reference to the provided `id` instead:
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

See the [E-commerce tutorial](https://docs.fauna.com/fauna/current/tutorials/ecommerce) for more info on modeling relational data with Fauna.

## Indexes
Retrieving one document at a time with `Get` won't get you very far. Since collections are just buckets of documents, you need to use indexes to create some order, retreive multiple documents, sort, filter, and more.

The combination of a collection and an index is probably the closest you will get to a table of a relational database. The big difference being that table rows can only belong to a single table. In contrast, Fauna indexes can have multiple collections as sources of data and documents can belong to as many indexes as you need.

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
* [Match](https://docs.fauna.com/fauna/current/api/fql/functions/match) combines the index with some optional parameters to feed `Paginate` with the necessary configuration to actually retrieve the matching data. At this point no actual data has been yet retrieved from Fauna.
* [Paginate](https://docs.fauna.com/fauna/current/api/fql/functions/paginate) takes the output of `Match` (actually a reference to a [Set](https://docs.fauna.com/fauna/current/api/fql/types#set)), retrieves data from Fauna, and finally returns a [Page](https://docs.fauna.com/fauna/current/api/fql/types#page) with an array of references.

Later on we'll see how to get actual documents instead of references and how to use `Match` for filtering.

#### Page size
By default `Paginate` returns pages with 64 items. You can define how many items you want to receive with `size`:
```js
Paginate(
  Match(Index('all_Fruits')),
  {size: 3}
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
Fauna gives us the next document with `after` to use as a cursor because in this case there are more results than fit in a single page of `3`.

See the [Paginate](https://docs.fauna.com/fauna/current/api/fql/functions/paginate) docs for more info on using cursors.

#### Unique values
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

We'll see what that `terms` object is later on.

**Note:** `["data", "email"]` represents a path. We'll see more about paths later but you can think of those as `data/email` or even `data.email`.

## Gettings multiple documents from references
Ok so we know how to get a list of references from an index. How do we actually get documents?

When programming you might have come across `map` which executes a function on each item of an array and returns a new array. For example in JavaScript we use [`myArray.map()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map).

Fauna also has a [Map](https://docs.fauna.com/fauna/current/api/fql/functions/index) function. We can use it to retrieve an array of documents from an array of references:
```js
Map(
  Paginate(Match(Index("all_Fruits"))),
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

We already understand from a previous example that `Paginate(Match(Index('all_Fruits')))` returns an array of references, right? So `Map` iterates over this array, executes `Lambda` on each item, and returns a new array with documents.

This is the tricky part: `Lambda("fruitRef", Get(Var("fruitRef")))`:
* [Lambda](https://docs.fauna.com/fauna/current/api/fql/functions/lambda) is used in FQL to define anonymous functions. In this example we are defining an function that will take a reference and return a document.
* `Lambda("fruitRef"...` defines a function parameter. It could be named anything: `fruit`, `X`, `Tarzan`, etc. The name is irrelevant. In this case the parameter will receive a single document reference that `Map` will pass to `Lambda` from `Paginate`.
* `Var("fruitRef")` evaluates the variable named `fruitRef` in the context of the function. You couldn't simply use `fruitRef` or `"fruitRef"` because FQL wouldn't know what do with it. [Var](https://docs.fauna.com/fauna/current/api/fql/functions/lambda) explicitly tells Fauna to find and evaluate a variable in the current context.
* Finally `Get` receives a reference from `Var` and returns an actual document. This document is returned by `Lambda` to `Map` to form an array of documents.

Consider this JavaScript example:
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
## Sorting results
To retrieve a list of ordered documents first we need to create a new index.
```js
CreateIndex({
  name: "all_Fruits_sorted_by_name",
  source: Collection("Fruits"),
  values: [
    { field: ["data", "name"] },
    { field: ["ref"] }
  ]
})
```
The new bit here is this `values` array.

We saw earlier that indexes also accept a `terms` array. The idea is simply that `terms` are **input** parameters for our index, and `values` are the actual values the index will **output**.

In this case the index will return two values for each result:
* `{ field: ["data", "name"] }` will return the `name` in the `data` part of the document.
* `{ field: ["ref"] }` will return a reference to the document.

Proof:
```js
Paginate(Match(Index("all_Fruits_sorted_by_name")))

// Result:

{
  "data": [
    [
      "Apple",
      Ref(Collection("Fruits"), "264476914201133588")
    ],
    [
      "Avocado",
      Ref(Collection("Fruits"), "264476977509958164")
    ],
    [
      "Banana",
      Ref(Collection("Fruits"), "264476925331767828")
    ],
    [
      "Lemon",
      Ref(Collection("Fruits"), "264477007396471316")
    ],
    // etc
  ]
}
```
As you can see the documents are already sorted by `name` alphabetically. Fauna will automatically sort the results by the fields you add to `values`.

If you want to reverse the order of the results you'd need to create a new index and use the `reverse` setting:
```js
CreateIndex({
  name: "all_Fruits_sorted_by_name_reverse",
  source: Collection("Fruits"),
  values: [
    { field: ["data", "name"], reverse: true },
    { field: ["ref"] }
  ]
})
```
Later on we'll see how to actually get a list of documents from these results.

## Filtering results
Filtering is similar to sorting and is done using indexes. Instead of adding `values` we need to add a `terms` array.

Remember: `terms` is input, `values` is output.

```js
CreateIndex({
  name: "Fruits_by_name",
  source: Collection("Fruits"),
  terms: [
    { field: ["data", "name"]}
  ]
})
```
So now we can use `Match` to pass a parameter to our index:
```js
Paginate(
  Match(Index("Fruits_by_name"), "Apple")
)

// Result:

{
  "data": [
    Ref(Collection("Fruits"), "264476914201133588")
  ]
}
```
Or if we want to get documents instead of references:
```js
Map(
  Paginate(Match(Index("Fruits_by_name"), "Apple")),
  Lambda("fruitRef", Get(Var("fruitRef")))
)

// Result:

{
  "data": [
    {
      "ref": Ref(Collection("Fruits"), "264476914201133588"),
      "ts": 1588483690410000,
      "data": {
        "name": "Apple"
      }
    }
  ]
}
```
If we know in advance we will get a single result we could simply use `Get` instead of `Paginate`:
```js
Get(Match(Index("Fruits_by_name"), "Apple"))

// Result:

{
  "ref": Ref(Collection("Fruits"), "264476914201133588"),
  "ts": 1588483690410000,
  "data": {
    "name": "Apple"
  }
}
```

If you want to know how to sort and filter at the same time check the [Search and sort with indexes tutorial](https://docs.fauna.com/fauna/current/tutorials/indexes/search_and_sort).

## About paths and using Select
Remember how we got an array of arrays when <a href="#sorting-results">sorting our data</a>?

To recap here's the ouput we have from the index:
```js
Paginate(Match(Index("all_Fruits_sorted_by_name")))

// Result:

{
  "data": [
    [
      "Apple",
      Ref(Collection("Fruits"), "264476914201133588")
    ],
    // etc
  ]
}
```
This is no bueno since we actually want documents and not references.

The problem we have now is that we cannot use `Lambda("fruitRef", Get(Var("fruitRef")))` like we did before to get a document. If you do that you will get an error because you wouldn't be passing a `Ref` to `Get`, you'd be actually passing this array:
```js
[
  "Apple",
  Ref(Collection("Fruits"), "264476914201133588")
]
```
Damn!

The solution is to find a way to select the second item of that array, which is a reference, and pass it to `Get` to retrieve a document.

We're going to use [Select](https://docs.fauna.com/fauna/current/api/fql/functions/select) for this:

```js
Map(
  Paginate(Match(Index("all_Fruits_sorted_by_name"))),
  Lambda("itemFromIndex", Get(Select([1], Var("itemFromIndex"))))
)

// Result:

{
  "data": [
    {
      "ref": Ref(Collection("Fruits"), "264476914201133588"),
      "ts": 1588483690410000,
      "data": {
        "name": "Apple"
      }
    },
    {
      "ref": Ref(Collection("Fruits"), "264476977509958164"),
      "ts": 1588483750760000,
      "data": {
        "name": "Avocado"
      }
    },
    // etc
  ]
}
```
This is the interesting bit `Select([1], Var("itemFromIndex"))`.

We already know from [a previous section](#gettings-multiple-documents-from-references) that `Var("itemFromIndex")` will return whatever `Lambda` is getting into its parameter which in this case is this:
```js
[
  "Apple",
  // 👇 we want this Ref to be able to get a document!
  Ref(Collection("Fruits"), "264476914201133588")
]
```
To do that we pass this path `[1]` to `Select`. This will return a `Ref` that we can use with `Get` and *voila!* we will have our document.

**Note:** Arrays in Fauna are zero based so we need to use the index `1` to select the second item from the array.

#### Selecting document properties
`Select` can also be used with paths that point to document properties (eg: `["data","name"]`) as we saw when <a href="#sorting-results">sorting</a> and <a href="#filtering-results">filtering</a> results.

To clarify this point further imagine we had a document with this structure:
```js
{
  data: {
    name: "Pier"
    country: {
      name: "Spain",
      city: {
        name: "Palma de Mallorca"
      }
    }
  }
}
```
To get the city name we'd use the following path: `["data", "country", "city", "name"]`.

## Getting custom results instead of full documents
Until now we've retrieved full documents but it may not make sense to do that every time we query Fauna.

Imagine we have a `Posts` collection with hundreds of posts such as this one:
```js
{
  data: {
    title: "Lorem ipsum",
    content: "This is the story about that time I took a latin class and...",
    tags: ["boring"],
    authorRef: Ref(Collection("Authors"), "264471980339626516"),
    status: 'DRAFT'
    // etc
  }
}
```
It would be a waste of bytes to send the complete post if we only wanted to present a list of posts to the user. You could certainly solve that in your backend logic but it can be solved in FQL too.

To do that we need to use [Let](https://docs.fauna.com/fauna/current/api/fql/functions/let) to create a custom expression.

By this point you should already know how to create collections, documents, and indexes. Create a `Posts` collection, add some posts to it, and then create an `all_Posts` index.

Done? Great.

This is how we use `Let` to retreive a custom expression from Fauna instead of full documents:
```js
Map(
  Paginate(Match(Index("all_Posts"))),
  Lambda("postRef", Let(
    {
      postDocument: Get(Var("postRef"))
    },
    {
      id: Select(["ref", "id"], Var("postDocument")),
      title: Select(["data", "title"], Var("postDocument"))
    }
  ))
)

// Result:

{
  "data": [
    {
      "id": "264538580516340242",
      "title": "Lorem ipsum"
    },
    {
      "id": "264538588626027026",
      "title": "Some other post"
    },
    {
      "id": "264538600151974418",
      "title": "And yet another boring post"
    },
    {
      "id": "264538616193090066",
      "title": "Lorem Ipsum the second"
    }
  ]
}
```
Boom shaka laka!

Let's break this down.

We already know what `Map`, `Paginate`, and `Lambda` are doing from previous sections.

This is the interesting bit:
```js
Lambda("postRef", Let(
  {
    postDocument: Get(Var("postRef"))
  },
  {
    id: Select(["ref", "id"], Var("postDocument")),
    title: Select(["data", "title"], Var("postDocument"))
  }
))
```
We are defining a `Lambda` function which will return whatever `Let` returns.

As you can see `Let` accepts two lists of pair values. These lists are called dictionnaries or objects in some programming languages.

The first list are variables you define to be used later on. In the [docs](https://docs.fauna.com/fauna/current/api/fql/functions/let) they call this first list `bindings`:
```
{
  postDocument: Get(Var("postRef"))
}
```
These bindings will be available in the context of our `Let`, and even other nested `Let`. Again, the name doesn't matter. It could be `post`, `X`, `whatever`, etc.

The value of `postDocument` is a document. Why? Because it's the result of `Get` which we know returns a document. `Var("postRef")` is evaluating the `postRef` variable which is the input parameter of our `Lambda`.

The second list of values is the **output** of `Let`, or the custom expression that `Let` will return:
```js
{
  id: Select(["ref", "id"], Var("postDocument")),
  title: Select(["data", "title"], Var("postDocument"))
}
```
Here we are defining `id` and `title`. Again, you can use any name you wish. In both cases we are selecting a property from the `postDocument` document by using `Select` as we saw in the previous section.

If you want to see a more complex `Let` example see [this Gist](https://gist.github.com/ptpaterson/82c01afc9b0ff624f96141a078b5ab54) by [ptpaterson](https://gist.github.com/ptpaterson).
