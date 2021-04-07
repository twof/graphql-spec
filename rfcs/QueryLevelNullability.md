# RFC: Query Level Nullability 

**Proposed by:** 
- [Liz Jakubowski](<social or github link here>) - Yelp
- [Alex Reilly](<social or github link here>) - Yelp

This RFC proposes creating a syntactical construct for client developers to 
express "nullability" in their queries.

## Defintions

Nullability: a concept that exists accross many programming lanugage (eg [Swift](https://developer.apple.com/documentation/swift/optional), [Kotlin](https://kotlinlang.org/docs/null-safety.html#nullable-types-and-non-null-types), [SQL](https://www.w3schools.com/sql/sql_notnull.asp))
that is used to express when users can be certain that a value can or can never be `null` 
(or the language equivilent). Nullability language constructs (eg `?` in Swift/Kotlin)
have become popular due to their ability to solve ergonomic problems in languages
such as those surrounding `NullPointerException` in Java.

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
as optional properties on the resulting type:
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
In many cases, the client should error if the business `name` is nil. If codegen were out of the picture,
we would be able to throw an error at JSON response-parsing time if it's missing, or otherwise instantiate
a hand-written business object with a non-optional `name` property. From that point on, all feature code
can happily assume that it has a non-nil business name to work with.

## 🧑‍💻 Proposed syntax
It would make more sense if the client could express that `name` must be non-nil _in the query itself_:
```graphql
query GetBusinessName($encid: String!) {
  business(encid: $encid) {
    name!  <-- this!
  }
}
```
On web where codegen is not used, the client no longer needs to handle the case where expected fields are missing.
On mobile platforms where codegen is used, clients have full control over the optionality of the properties on the
generated types. Since optionality is expressed in the query rather than the schema, it's flexible enough to accommodate
various use-cases (e.g., where the business `name` _is_ allowed to be optional).

## Implementation
https://github.yelpcorp.com/wxue/graphql-js

### // EVERYTHING BELOW THIS POINT IS NOT RELATED TO THE WIP RFC. IT IS BEING USED
### // AS A TEMPLATE

### Use cases

1. A GraphQL server wants to **log how often each field in the schema is
   requested**. This may be implemented by incrementing a counter by the name of
   the schema coordinate for each field executed in a request.

   _Existing implementations: Yelp (internal), Facebook (internal),
   [Shopify (API health report)][shopify-api-health]_

   [shopify-api-health]: https://shopify.dev/concepts/about-apis/versioning/api-health

1. GraphiQL and other playgrounds / documentation sites want to show a list of
   **search results** when a user searches for a type or field name. We can
   display a list of schema coordinates that match the search term. A schema
   coordinate can also be used in the hyperlink to form a permalink for
   documentation for a particular field.

   _Existing implementations: GraphiQL, Apollo Studio (see "Prior Art")_

1. A developer may want to perform **analytics** on all known
   [persisted queries][apq] - e.g. what are the most commonly used fields across
   all documents. Schema coordinates may be used as the index/lookup keys when
   storing this information in the database.

   _Existing implementations: Yelp (internal)_

   [apq]: https://www.apollographql.com/docs/apollo-server/performance/apq/

1. A **GitHub bot** may want to warn developers in a Pull Request comment
   whenever the schema diff contains a breaking change. Schema coordinates can be
   used to provide a list of which fields were broken.

   _Existing implementations: GraphQL Inspector (see "Prior Art")_

1. **GraphQL IDEs** (e.g. GraphiQL, GraphQL Playground, Apollo Studio) may wish
   to display the schema definition type of a node in a query when hovering over
   it.

   <details>
   <summary>Example</summary>
   ![](https://i.fluffy.cc/g78sJCjCJ0MsbNPhvgPXP46Kh9knBCKF.png)
   </details>

   Schema coordinates can be used to form the left hand side of this popover.

_Existing implementations: Apollo Studio (see "Prior Art")_

## ✅ RFC Goals

- There be one, unambiguous way to write a "schema coordinate" that refers to a
  particular element in a GraphQL schema. (This is to avoid users having to
  "simplify" more complex coordinates to produce a canonical representation.)
- Schema coordinate syntax should build off of existing de-facto standards
  already adopted for this purpose (i.e. `Foo.bar`)
- Schema coordinate syntax is open for extension in the future. We should make
  design choices that give us flexibility and anticipate future syntax needs
  (based off of discussions around this RFC).

## 🚫 RFC Non-goals

- This does not cover "selectors" or "wildcard" syntax - e.g. `User.*`. _(See
  alternatives considered.)_
- There are **no proposed GraphQL language/syntax changes**
- There are **no proposed GraphQL runtime changes**
- [Schema coordinate non-goals](#-syntax-non-goals)

## 🧑‍💻 Proposed syntax

### `Type`

Refers to a named type (e.g. something represented by `__typename` in a GraphQL
introspection call).

### `Type.attribute`

Refers to a named attribute on the named type.

Not all types support this. For object types and interface types this is a field,
for input objects this would be an input field, for enums this would be an enum
value, for future GraphQL types this will relate to a related concept if they
have one (e.g. for the [proposed "tagged" type][tagged-type] it would refer to
the "member field").

[tagged-type]: https://github.com/graphql/graphql-spec/pull/733

### `Type.field(argName:)`

Refers to a named argument on the named field of the named type.

### `@directive`

References the given named directive

### `@directive(argName:)`

References the named argument of the named directive.

### ✨ Examples

For example, consider the following schema:

```graphql
directive @private(scope: String!) on FIELD

type Person {
  name: String
  email: String @private(scope: "loggedIn")
}

type Business {
  name: String
  owner: Person
}

type Query {
  searchBusinesses(name: String): [Business]
}
```

We can write the following schema coordinates:

- `Person` uniquely identifies the the "Person" type
- `Business` uniquely identifies the the "Business" type
- `Person.name` uniquely identifies the "name" field on the "Person" type
- `Business.name` uniquely identifies the "name" field on the "Business"
  type
- `Business.owner` uniquely identifies the "owner" field on the "Business" type
- `Query.searchBusinesses` uniquely identifies the "searchBusinesses" field on
  the "Query" type
- `Query.searchBusinesses(name:)` uniquely identifies the "name" argument on the
  "searchBusinesses" field on the "Query" type
- `@private` uniquely identifies the "private" directive
- `@private(scope:)` uniquely identifies the "scope" argument on the "private"
  directive

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

## 🥣 Document -> Schema Coordinate serialization

Use cases 3 and 5 above imply that a mapping from GraphQL query nodes to schema
coordinates is performed.

For example, consider the following schema:

```graphql
type Person {
  name: String
}

type Business {
  name: String
  owner: Person
}

type Query {
  searchBusiness(name: String): [Business]
}
```

And the following query:

```graphql
query {
  searchBusinesses(name: "El Greco Deli") {
    name
    owner {
      name
    }
  }
}
```

From the query above, we may calculate the following list of schema coordinates:

- `Query.searchBusinesses`
- `Business.name`
- `Business.owner`
- `Person.name`

`Query.searchBusinesses(name:)` is also a valid member of the output set. The
serialization algorithm may optionally choose to output all permutations of field
arguments used, should this be specified.

A library has been written to demonstrate this mapping:
<https://github.com/sharkcore/extract-schema-coordinates>.

## 🗳️ Alternatives considered

### Naming

- **"Schema Selectors"**

  "Selectors" is a term used in [HTML](https://www.w3.org/TR/selectors-api/) and
  [CSS](https://drafts.csswg.org/selectors-4/) to _select_ parts of an HTML
  document.

  This would be a compelling, familiar choice - however, we've decided to not
  support wildcard expansion in this spec. See the section
  [Syntax Non-goals](#-syntax-non-goals).

- **"type/field pairs"**

  This was the original working name. However, there already exists more
  established terminology for this concept, and we also wish to describe more
  than just types on fields.

- **"Field Coordinates"**

  "Field Coordinates" is already understood and used by the popular
  [GraphQL Java](https://github.com/graphql-java/graphql-java) project.

  [Feedback in the August GraphQL Working Group meeting](https://youtu.be/FYF15RA9H3k?t=3786)
  hinted that since we're targeting also describing arguments, _field_
  coordinates might not be the right name. Hence "Schema Coordinates" is chosen
  instead, as a more generalized form of this.

- **"GraphQL Coordinates"**

  Similar to Field Coordinates/Schema Coordinates - however, "GraphQL
  Coordinates" is potentially ambiguous as to if it describes _schema_ members,
  _query/document_ members or response object members.

- **"Field path" / "GraphQL path"**

  [`path` exists as an attribute on `GraphQLResolveInfo`](https://github.com/graphql/graphql-js/blob/8f3d09b54260565/src/type/definition.js#L951).

  Given the following query:

  ```graphql
  query {
    searchBusinesses(name: "El Greco Deli") {
      name
      owner {
        name
      }
    }
  }
  ```

  `Person.name` in the response may be written as the following "field path":

  ```json
  ["query", "searchBusinesses", 1, "owner", "name"]
  ```

  Note that here, the "path" is a serialized _response_ tree traversal, instead
  of describing the location of the field in the _schema_.

  Since "path" is already used in GraphQL nomenclature to describe the location
  of a field in a response, we'll avoid overloading this term.

### Separator

This RFC proposes using "`.`" as the separator character between a type and
field. The following have also been proposed:

- `Foo::bar`
- `Foo#bar`
- `Foo->bar`
- `Foo~bar`
- `Foo:bar`

"`.`" is already used in the existing implementations of field coordinates, hence
the suggested usage in this RFC. However, we may wish to consider one of the
alternatives above, should this conflict with existing or planned language
features.

### Field Arguments

We have discussed multiple options for selecting arguments on fields. ([PR][pr],
and [December WG Meeting][wg-meeting]). For example, consider the following
schema:

[pr]: https://github.com/graphql/graphql-spec/pull/746#discussion_r526243627
[wg-meeting]: https://youtu.be/Duh4MRXQRQA?t=2506

```graphql
type Query {
  rollDice(numDice: Int, numSides: Int): Int
}
```

We may want to refer to the `numDice` argument in a schema selector. Two options
for this syntax are:

1. `Query.rollDice.numDice`
1. `Query.rollDice(numDice:)`

#### Pros for `Query.rollDice.numDice`

- Less bytes/characters to type
- May allow for extension to include nested "field paths" (e.g. Foo.bar.Baz.qux...)
- [Prior usage][graphiql-usage] of this syntax to represent state internally

[graphiql-usage]: https://github.com/graphql/graphql-spec/pull/746#issuecomment-752941039

#### Pros for `Query.rollDice(numDice:)`

- Indicating arguments with colons disambiguates against other types of schema
  nodes. For those unfamiliar with schema selectors, it may be unclear if the
  third dot separated item refers to a directive or a child object etc.
- Using trailing colons for arguments is borrowed from other languages (e.g.
  [Swift][swift]). This may indicate to users who are unfamiliar with schema
  coordinates, but recognize this from other languages, that `numDice:` refers
  to an argument. The function parentheses and colons more strongly communicate
  "this is an argument!" than a second dot separator.

#### Decision

We are choosing `Query.rollDice(numDice:)` to optimize for **readability** and
**extensibility**.

Given our expected use cases, we assume Schema Coordinates will be _read_ more
often than they are _written_ (e.g. error messages in a stack trace from a
schema linting tool). Readers may be unfamiliar with its syntax. We want to
"hint" as much as possible the meaning of the coordinates in its syntax. We
think `(numDice:)` more clearly communicates that "numDice" is an argument, over
`.numDice`.

In addition, we want to be mindful of extensions to this syntax in the future.
Using dots only as a separator may overload the meaning of elements in schema
coordinates in the future. (If we capture new schema node types, or nested
paths.)

> We should make sure that the spec enables future innovation including using it
> for things other than schema coordinates. To my mind the (foo:) syntax is more
> flexible in this regard. For example, I can imagine referring to:
>
> 1. `Foo.bar(baz.qux:)`: the qux field of the input object referred to from the
>    baz argument of the bar field on the Foo type.
> 2. `Foo.bar(baz:).qux`: the qux field on the return type of the bar field
>    (with baz: argument) of the Foo type.
> 3. `Foo.bar.baz.qux`: the qux field of the return type of the baz field on the
>    return type of the bar field on type Foo.
>
> If we were to only use periods then all of these would come out the same as
> `Foo.bar.baz.qux`, and this ambiguity precludes this kind of reusal of the
> schema-coordinates syntax for this use case (which is outside the scope of the
> schema coordinates spec, for sure, but is still a potential use-case for the
> syntax).
>
> ~ [benjie](https://github.com/graphql/graphql-spec/pull/746#discussion_r527639917)

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

## Answered questions

- **Would we want to add a method to graphql-js?** A `fieldCoordinateToFieldNode`
  method (for example) may take in a field coordinate string and return a field
  AST node to serve as a helper / reference implementation of the algorithm to
  look up the field node.

  _Update:_ [This was discussed in the August Working Group Meeting][meeting] -
  it was suggested to keep any utilities as third party libraries to avoid edge
  ambiguity problems, and to be able to iterate faster.

  [meeting]: https://youtu.be/FYF15RA9H3k?t=2865
