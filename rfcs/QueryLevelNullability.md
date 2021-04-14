# RFC: Query Level Nullability 

**Proposed by:** 
- [Liz Jakubowski](https://github.com/lizjakubowski) - Yelp iOS
- [Sanae Rosen](<social or github link here>) - Yelp Android
- [Mark Larah](https://github.com/magicmark) - Yelp Web
- [Alex Reilly](<social or github link here>) - Yelp iOS

This RFC proposes creating a syntactical construct for client developers to 
express "nullability" in their queries.

## Definitions

Nullability: A concept that exists across many programming languages (eg [Swift](https://developer.apple.com/documentation/swift/optional), [Kotlin](https://kotlinlang.org/docs/null-safety.html#nullable-types-and-non-null-types), [SQL](https://www.w3schools.com/sql/sql_notnull.asp))
that is used to express when users can be certain that a value can or can never be `null` 
(or the language equivalent). Nullability language constructs (eg `?` in Swift/Kotlin)
have become popular due to their ability to solve ergonomic problems in languages
such as those surrounding `NullPointerException` in Java.

Codegen: Short for Code Generation, tools that exist for working with GraphQL which accept a schema and a set of documents as
input, and output code in a language of choice that represents the data returned by those 
operations. GraphQL codegen tools exist for many platforms. As an example, [here's some info about Apollo's offerings](https://github.com/apollographql/apollo-tooling#code-generation).

## 📜 Problem Statement

GraphQL is a "nullable by default" language meaning that all properties are allowed to be `null`.
This is in contrast to the two modern languages used on mobile clients, Swift and Kotlin,
which are both non-null by default. In Swift and Kotlin, unless developers otherwise
specify it, properties cannot be `null`.

This mismatch creates some dissonance for developers who are currently forced to deal with
problems that commonly surround nullablility. This makes large numbers of null fields tedious and
time-consuming to work with.

It is often unclear how to handle partial results.
- What elements are essential to having an adequate user experience? How many elements can fail before you 
  should just show an error message instead?
- When any combination of elements can fail, it is very hard to anticipate all edge cases and how the app 
  might look and work, including transitions to other screens.

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
Semantically the GraphQL `!` operator is nearly identical to its counterpart in Swift (also represented by `!`) which is
referred to as the "force unwrap operator". In Swift, for example, you can cast a string to an integer with `Int("5")` 
but the string being cast may not be a valid number, so that statement will return `null` rather than an integer if the
string cannot be turned into an integer. If you want to ensure that the statement does not return `null` you can instead 
write `Int("5")!`. If you do that, an exception will be thrown if the statement would return `null`.

In GraphQL, the `!` operator will act similarly. In the case that `name` does not exist, the query will return an
error to the client rather than any data.

On web where codegen is not used, the client no longer needs to handle the case where expected fields are missing.
On mobile platforms where codegen is used, clients have full control over the nullability of the properties on the
generated types. Since nullability is expressed in the query rather than the schema, it's flexible enough to accommodate
various use-cases (e.g., where the business `name` _is_ allowed to be nullable).

### `!`

We have chosen `!` because `!` is already being used in the GraphQL spec to indicate that a field is non-nullable.
Incidentally the same precedent exists in Swift (`!`) and Kotlin (`!!`) which both use similar syntax to indicate
"throw an exception if this value is `null`". 

### Use cases

#### When a field is necessary to the function of the client
There are times where a field can be `null`, but a feature that queries for the field will not function
if it is `null`. For example if you are trying to render an information page for a movie, you won't
be able to do that if the name field of the movie is missing. In that case it would be preferable
to fail as early as possible.

### ✨ Examples

#### Non-nullable field
```graphql
query GetBusinessName($encid: String!) {
  business(encid: $encid) {
    name!
  }
}
```
would codegen to the following type on iOS.

```swift
struct GetBusinessNameQuery.Data.Business {
  let name: String // lack of `?` indicates that `name` will never be `null`
}
```

#### Non-nullable object with nullable fields
```graphql
query {
  business(id: 4) {
    reviews! {
      rating
      text
    }
  }
}
```

## ✅ RFC Goals
- Non-nullable syntax that is based off of syntax that developers will already be familiar with

## 🚫 RFC Non-goals

## 🗳️ Alternatives considered

### Make Schema Fields Non-Nullable Instead
Discussion on [this topic can be found here](https://medium.com/@calebmer/when-to-use-graphql-non-null-fields-4059337f6fc8)
// We definitely want to flesh this out

### Use a directive such as `@non-null` instead
This is the alternative being used at some of the companies represented in this proposal for the time being.
However this feels like a common enough need to call for a language feature. A single language feature
rather than using directives as a workaround would enable more unified public tooling around GraphQL.

### Write wrapper types that null-check fields
This is the alternative being used at some of the companies represented in this proposal for the time being.
It's quite labor intensive and the work is quite rote. It more or less undermines the purpose of
having code generation.

### Alternatives to `!`
#### `!!`
This would follow the precedent set by Kotlin.

### Make non-nullability apply recursively
For example, everything in this tree would be non-nullable
```graphql
query! {
  business(id: 4) {
    name
  }
}
```
// TODO: This was rejected when it was proposed. Looking for more info as to why

## 🙅 Syntax Non-goals

This syntax consciously does not cover the following use cases:

- **Default Values**
  The syntax being used in this proposal causes queries to error out in the case that
  a `null` is found. As an alternative, some languages provide syntax (eg `??` for Swift)
  that says "if a value would be `null` make it some other value instead". We are
  not interested in covering that in this proposal.
  
## Implementation
GraphQL.js: https://github.yelpcorp.com/wxue/graphql-js
