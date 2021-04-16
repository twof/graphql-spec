# RFC: Query Level Nullability 

**Proposed by:**

- [Alex Reilly](<social or github link here>) - Yelp iOS
- [Liz Jakubowski](https://github.com/lizjakubowski) - Yelp iOS
- [Mark Larah](https://github.com/magicmark) - Yelp Web
- [Sanae Rosen](<social or github link here>) - Yelp Android
- [Wei Xue](<social or github link here>) - Yelp iOS

This RFC proposes a syntactical construct for GraphQL clients to express the **nullability** of schema fields requested
in a query.

## Definitions

- **Nullability**. A feature of many programming languages (eg [Swift](https://developer.apple.com/documentation/swift/optional),
  [Kotlin](https://kotlinlang.org/docs/null-safety.html#nullable-types-and-non-null-types), [SQL](https://www.w3schools.com/sql/sql_notnull.asp))
  that is used to indicate whether or not a value can be `null`.

  Nullability language constructs (e.g. `?` in Swift/Kotlin) have become popular due to their ability to solve ergonomic
  problems in programming languages, such as those surrounding `NullPointerException` in Java.

- **Codegen**. Short for "code generation", in this proposal refers to tools that generate code to facilitate using
GraphQL on the client. GraphQL codegen tooling exists for many platforms:
  - [Apollo](https://github.com/apollographql/apollo-tooling#code-generation) has a code generator for Android (Kotlin)
    and iOS (Swift) clients
  - [The Guild](https://www.graphql-code-generator.com/) has a TypeScript code generator for web clients

  GraphQL codegen tools typically accept a schema and a set of documents as input, and output code in a language of
  choice that represents the data returned by those operations.

  For example, the Apollo iOS codegen tool generates Swift types to represent each query document, as well as model types
  representing the data returned from those queries. Notably, a nullable field in the schema becomes an `Optional`
  property on the generated Swift model type, represented by `?` following the type name.

  In the example below, the `Business` schema type has a nullable field called `name`.
  ```graphql
  # Schema
  type Business {
    # The unique identifier for the business (non-nullable)
    id: String!
  
    # The name of the business (nullable)
    name: String
  }

  # Document
  query GetBusinessName($id: String!) {
    business(id: $id) {
      name
    }
  }
  ```
  At build time, Apollo generates the following Swift code (note: the code has been shortened for clarity).
  ```swift
  struct GetBusinessNameQuery {
    let id: String

    struct Data {
      let business: Business?

      struct Business {
        /// Since the `Business.name` schema field is nullable, the corresponding codegen Swift property is `Optional`
        let name: String?
      }
    }
  }
  ```
  The query can then be fetched, and the resulting data handled, as follows:
  ```swift
  GraphQL.fetch(query: GetBusinessNameQuery(id: "foo"), completion: { result in
    guard case let .success(gqlResult) = result, let business = gqlResult.data.business else { return }

    // Often, the client needs to provide a default value in case `name` is `null`.
    print(business?.name ?? "null")
  }
  ```

## 📜 Problem Statement

The two modern languages used on mobile clients, Swift and Kotlin, are both non-null by default. By contrast, in a
GraphQL type system, every field is nullable by default. From the [official GraphQL best practice](https://graphql.org/learn/best-practices/#nullability):

> This is because there are many things that can go awry in a networked service backed by databases and other
> services. A database could go down, an asynchronous action could fail, an exception could be thrown. Beyond
> simply system failures, authorization can often be granular, where individual fields within a request can
> have different authorization rules.

This discrepancy means that developers using strongly-typed languages must perform `null` checks anywhere that GraphQL
codegen types are used, significantly decreasing the benefits of codegen. In some cases, the codegen types may be so
complex that a new model type which encapsulates the `null` handling is written manually.

While the schema can have nullable fields for valid reasons (such as federation), in some cases the client wants to decide
if it accepts a `null` value for the result to simplify the client-side logic. In addition, a syntax for this concept
would allow codegen tooling to generate model types that are more ergonomic to work with, since the fields would have the
desired nullability.

## 🧑‍💻 Proposed syntax

The client can express that a schema field is required using the `!` syntax:
```graphql
query GetBusinessName($encid: String!) {
  business(encid: $encid) {
    name!
  }
}
```
Semantically the GraphQL `!` operator is nearly identical to its counterpart in Swift (also represented by `!`) which is
referred to as the "force unwrap operator".

In Swift, for example, you can cast a string to an integer with `Int("5")` but the string being cast may not be a valid
number, so that statement will evaluate to `null` rather than an integer if the string cannot be cast to an integer. If
you want to ensure that the statement does not return `null` you can instead write `Int("5")!`. If you do that, an
exception will be thrown if the statement would evaluate to `null`.

In GraphQL, the `!` operator will act similarly. In the case that `name` does not exist, the query will return an error
to the client rather than any data.

### `!`

We have chosen `!` because `!` is already being used in the GraphQL spec to indicate that a field is non-nullable.
Incidentally the same precedent exists in Swift (`!`) and Kotlin (`!!`) which both use similar syntax to indicate
"throw an exception if this value is `null`".

### Use cases

#### When a field is necessary to the function of the client

Expressing nullability in the query, as opposed to the schema, offers the client more flexibility and control over
whether or not an error is thrown.

There are cases where a field is `nullable`, but a feature that fetches the field will not function if it is `null`.
For example if you are trying to render an information page for a movie, you won't be able to do that if the name field
of the movie is missing. In that case it would be preferable to fail as early as possible.

According to the official GraphQL best practice, this field should be non-nullable:

> When designing a GraphQL schema, it's important to keep in mind all the problems that could go wrong and if "null" is
> an appropriate value for a failed field. Typically it is, but occasionally, it's not. In those cases, use non-null
> types to make that guarantee.

However, the same field may not be "vital" for every feature in the application that fetches it. Marking the field as
non-null in the schema would result in those other features erroring unnecessarily whenever the field is "null".

#### When handling partial results

It is often unclear how to handle partial results.

- What elements are essential to having an adequate developer experience? How many elements can fail before you 
  should just show an error message instead?
- When any combination of elements can fail, it is very hard to anticipate all edge cases and how the UI 
  might look and work, including transitions to other screens.

Implementing these decisions significantly complicates the client-side logic for handling query results.

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
struct GetBusinessNameQuery {
  struct Data {
    struct Business {
      /// Lack of `?` indicates that `name` will never be `null`
      let name: String
    }
  }
}
```

#### Non-nullable object with nullable fields
```graphql
query GetBusinessReviews {
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
- Enable GraphQL codegen tools to generate more ergonomic types

## 🚫 RFC Non-goals

## 🗳️ Alternatives considered

### An official `@nonNull` directive

This solution offers the same benefits as the proposed solution. Since many GraphQL codegen tools already support the `@skip` and `@include` directives, this solution likely has a faster turnaround.

### Use a custom `@nonNull` directive

This is an alternative being used at some of the companies represented in this proposal for the time being.

While this solution simplifies some client-side logic, it does not meaningfully improve the developer experience for clients that rely on codegen, since codegen types typically cannot be customized based on a custom directive.

This feels like a common enough need to call for a language feature. A single language feature would enable more unified public tooling around GraphQL.

### Make Schema Fields Non-Nullable Instead
Discussion on [this topic can be found here](https://medium.com/@calebmer/when-to-use-graphql-non-null-fields-4059337f6fc8)

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

## 🙅 Syntax Non-goals

This syntax consciously does not cover the following use cases:

- **Default Values**
  The syntax being used in this proposal causes queries to error out in the case that
  a `null` is found. As an alternative, some languages provide syntax (eg `??` for Swift)
  that says "if a value would be `null` make it some other value instead". We are
  not interested in covering that in this proposal.
  
## Implementation
GraphQL.js: https://github.com/graphql/graphql-js/pull/2824
