# RFC: Query Level Nullability 

**Proposed by:** 
- [Liz Jakubowski](<social or github link here>) - Yelp
- [Alex Reilly](<social or github link here>) - Yelp

This RFC proposes creating a syntactical construct for client developers to 
express "nullability" in their queries.

## Defintions

Nullability: A concept that exists accross many programming lanugage (eg [Swift](https://developer.apple.com/documentation/swift/optional), [Kotlin](https://kotlinlang.org/docs/null-safety.html#nullable-types-and-non-null-types), [SQL](https://www.w3schools.com/sql/sql_notnull.asp))
that is used to express when users can be certain that a value can or can never be `null` 
(or the language equivilent). Nullability language constructs (eg `?` in Swift/Kotlin)
have become popular due to their ability to solve ergonomic problems in languages
such as those surrounding `NullPointerException` in Java.

Codegen: Short for Code Generation, tools that exist for working with GraphQL which take queries as
input and output code in a language of choice that represents data that will result from those 
queries. Codegen tools exist for many platforms. As an example, [here's some info about Apollo's offerings.](https://github.com/apollographql/apollo-tooling#code-generation)

## 📜 Problem Statement

GraphQL is a "nullable by default" language meaning that all properties are allowed to be `null`.
This is in contrast to the two modern languages used on mobile clients, Swift and Kotlin,
which are both non-null by default languages. In Swift and Kotlin, unless developers otherwise
specify it, properties cannot be `null`.

This mismatch creates some dissonance for developers who are currently forced into dealing the
problems that commonly surround nullablility in codebases that otherwise do not need to deal with
those problems.

In Yelp's GraphQL schema, almost all object fields are nullable except for those with ID type. 
This adheres to what seems to be the [official best practice](https://graphql.org/learn/best-practices/#nullability).

This poses a problem for the mobile clients that use Apollo's codegen feature. The codegen provides 
Swift/Kotlin types to represent queries used in the app. Nullable fields in the schema are represented
as nullable properties on the resulting type, represented by `?` following the type name:
```graphql
query GetBusinessName($encid: String!) {
  business(encid: $encid) {
    name
  }
}
```
```swift
struct GetBusinessNameQuery.Data.Business {
  let name: String?
}
```
In many cases, the client should error if the business `name` is null. If codegen were out of the picture,
we would be able to throw an error at JSON response-parsing time if it's missing, or otherwise instantiate
a hand-written business object with a non-nullable `name` property. From that point on, all feature code
can happily assume that it has a non-null business name to work with.

## 🧑‍💻 Proposed syntax

It would make more sense if the client could express that `name` must be non-null _in the query itself_:
```graphql
query GetBusinessName($encid: String!) {
  business(encid: $encid) {
    name!  <-- this!
  }
}
```
On web where codegen is not used, the client no longer needs to handle the case where expected fields are missing.
On mobile platforms where codegen is used, clients have full control over the nullability of the properties on the
generated types. Since nullability is expressed in the query rather than the schema, it's flexible enough to accommodate
various use-cases (e.g., where the business `name` _is_ allowed to be nullable).

In the case that a field decorated with `!` is `null`, the server is expected to return an error to the client.

### `!`

We have chosen `!` because `!` is already being used in the GraphQL spec to indicate that a field is non-nullable.
Incidentally the same precedent exists in Swift (`!`) and Kotlin (`!!`) which both use similar syntax to indicate
"throw an exception if this value is `null`". 

### Use cases

### ✨ Examples

```graphql
query GetBusinessName($encid: String!) {
  business(encid: $encid) {
    name!  <-- this!
  }
}
```
would codegen to the following type on iOS.

```swift
struct GetBusinessNameQuery.Data.Business {
  let name: String // lack of `?` indicates that `name` will never be `null`
}
```

## ✅ RFC Goals
- Non-nullable syntax that is based off of syntax that developers will already be familiar with

## 🚫 RFC Non-goals

## 🗳️ Alternatives considered

### Make Schema Fields Non-Nullable Instead
Discussion on [this topic can be found here](https://medium.com/@calebmer/when-to-use-graphql-non-null-fields-4059337f6fc8)
// We definitely want to flesh this out

### Alternatives to `!`
#### `!!`
This would follow the precedent set by Kotlin.

## Implementation
https://github.yelpcorp.com/wxue/graphql-js

### // EVERYTHING BELOW THIS POINT IS NOT RELATED TO THE WIP RFC. IT IS BEING USED
### // AS A TEMPLATE

## 🎨 Prior art

- The name "schema coordinates" is inspired from [GraphQL Java](https://github.com/graphql-java/graphql-java)
  (4.3k stars), where "field coordinates" are already used in a similar way as
  described in this RFC.

  - [GitHub comment](https://github.com/graphql/graphql-spec/issues/735#issuecomment-646979049)
  - [Implementation](https://github.com/graphql-java/graphql-java/blob/2acb557474ca73/src/main/java/graphql/schema/FieldCoordinates.java)

- GraphiQL displays schema coordinates in its documentation search tab:

  ![](https://i.fluffy.cc/5Cz9cpwLVsH1FsSF9VPVLwXvwrGpNh7q.png)

- [GraphQL Inspector](https://github.com/kamilkisiela/graphql-inspector) (840
  stars) shows schema coordinates in its output:

  ![](https://i.imgur.com/HAf18rz.png)

- [Apollo Studio](https://www.apollographql.com/docs/studio/) shows schema
  coordinates when hovering over fields in a query:

  ![](https://i.fluffy.cc/g78sJCjCJ0MsbNPhvgPXP46Kh9knBCKF.png)



## 🙅 Syntax Non-goals

This syntax consciously does not cover the following use cases:

- **Wildcard selectors**

  Those familiar with `document.querySelector` may be expecting the ability to
  pass "wildcards" or "star syntax" to be able to select multiple schema
  elements. This implies multiple ways of _selecting_ a schema node.

  For example, `User.address` and `User.a*` might both resolve to `User.address`.
  But `User.a*` could also ambiguously refer to `User.age`.

  It's unclear how wildcard expansion would work with respect to field
  arguments\*, potentially violating the requirement of this schema to _uniquely_
  identify schema components.

  \* _(e.g. does `Query.getUser` also select all arguments on the `getUser`
  field? Who knows! A discussion for another time.)_

  A more general purpose schema selector language could be built on top of this
  spec - however, we'll consider this **out of scope** for now.

- **Nested field paths**

  This spec does _not_ support selecting schema members with a path from a root
  type (e.g. `Query`).

  For example, given this schema

  ```graphql
  type User {
    name: String
    bestFriend: User
  }

  type Query {
    userById(id: String): User
  }
  ```

  The following are invalid schema coordinates:

  - `Query.userById.name`
  - `User.bestFriend.bestFriend.bestFriend.name`

  This violates a non-goal that there be one, unambiguous way to write a
  schema coordinate to refer to a schema member. Both examples can be
  "simplified" to `User.name`, which _is_ a valid schema coordinate.

  Should a use case for this arise in the future, a follow up RFC may investigate
  how schema coordinates could work with "field paths" (e.g. `["query", "searchBusinesses", 1, "owner", "name"]`) to cover this.

- **Directive applications**

  This spec does _not_ support selecting applications of directive.

  For example:

  ```graphql
  directive @private(scope: String!) on FIELD

  type User {
    name: String
    reviewCount: Int
    friends: [User]
    email: String @private(scope: "loggedIn")
  }
  ```

  You _can_ select the definition of the `private` directive and its arguments
  (with `@private` and `@private(scope:)` respectively), but you cannot select the
  application of the `@private` on `User.email`.

  For the stated use cases of this RFC, it is more likely that consumers want to
  select and track usage and changes to the definition of the custom directive
  instead.

  If we _did_ want to support this, a syntax such as `User.email@private[0]`
  could work. (The indexing is necessary since [multiple applications of the same
  directive is allowed][multiple-directives], and each is considered unique.)

  [multiple-directives]: http://spec.graphql.org/draft/#sec-Directives-Are-Unique-Per-Location

- **Union members**

  This spec does not support selecting members inside a union definition.

  For example:

  ```graphql
  type Breakfast {
    eggCount: Int
  }

  type Lunch {
    sandwichFilling: String
  }

  union Meal = Breakfast | Lunch
  ```

  You may select the `Meal` definition (as "`Meal`"), but you may **not** select
  members on `Meal` (e.g. `Meal.Breakfast` or `Meal.Lunch`).

  It is unclear what the use case for this would be, so we won't (yet?) support
  this. In such cases, consumers may select type members directly (e.g. `Lunch`).
