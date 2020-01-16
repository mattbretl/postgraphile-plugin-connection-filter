[![Package on npm](https://img.shields.io/npm/v/postgraphile-plugin-connection-filter.svg)](https://www.npmjs.com/package/postgraphile-plugin-connection-filter) [![CircleCI](https://circleci.com/gh/graphile-contrib/postgraphile-plugin-connection-filter.svg?style=svg)](https://circleci.com/gh/graphile-contrib/postgraphile-plugin-connection-filter)

# postgraphile-plugin-connection-filter
This plugin adds a `filter` argument for advanced filtering of Connections.

> **Warning:** Use of this plugin (particularly with the default options) may make it **astoundingly trivial** for a malicious actor (or a well-intentioned application that generates complex GraphQL queries) to overwhelm your database with expensive queries. See the Performance and Security section below for details.

## Performance and Security

By default, this plugin:
- Exposes a large number of filter operators, including some that can perform expensive pattern matching.
- Allows filtering on [computed columns](https://www.graphile.org/postgraphile/computed-columns/), which can result in expensive operations.
- Allows filtering on functions that return `setof`, which can result in expensive operations.
- Allows filtering on List fields (Postgres arrays), which can result in expensive operations.

To protect your server, you can:
- Use the `connectionFilterAllowedFieldTypes` and `connectionFilterAllowedOperators` options to limit the filterable fields and operators exposed through GraphQL.
- Set `connectionFilterComputedColumns: false` to prevent filtering on [computed columns](https://www.graphile.org/postgraphile/computed-columns/).
- Set `connectionFilterSetofFunctions: false` to prevent filtering on functions that return `setof`.
- Set `connectionFilterArrays: false` to prevent filtering on List fields (Postgres arrays).

Also see the [Production Considerations](https://www.graphile.org/postgraphile/production) page of the official PostGraphile docs, which discusses query whitelisting.

## Getting Started

### CLI

```bash
yarn add postgraphile
yarn add postgraphile-plugin-connection-filter
npx postgraphile --append-plugins postgraphile-plugin-connection-filter
```

### Library

```js
const express = require("express");
const { postgraphile } = require("postgraphile");
const ConnectionFilterPlugin = require("postgraphile-plugin-connection-filter");

const app = express();

app.use(
  postgraphile(process.env.DATABASE_URL, "app_public", {
    appendPlugins: [ConnectionFilterPlugin],
    graphiql: true,
  })
);

app.listen(5000);
```

## Handling `null` and empty objects

By default, this plugin will throw an error when `null` literals or empty objects (`{}`) are included in `filter` input objects. This prevents queries with ambiguous semantics such as `filter: { field: null }` and `filter: { field: { equalTo: null } }` from returning unexpected results. For background on this decision, see https://github.com/graphile-contrib/postgraphile-plugin-connection-filter/issues/58.

To allow `null` and `{}` in inputs, use the `connectionFilterAllowNullInput` and `connectionFilterAllowEmptyObjectInput` options documented under [Plugin Options](https://github.com/graphile-contrib/postgraphile-plugin-connection-filter#plugin-options). Please note that even with `connectionFilterAllowNullInput` enabled, `null` is never interpreted as a SQL `NULL`; fields with `null` values are simply ignored when resolving the query.

## Operators

The following operators are exposed by default, depending on the field type:

### Scalars

All of the scalar types generated by PostGraphile (BigFloat, BigInt, BitString, Boolean, Date, Datetime, Float, Int, InternetAddress, Interval, JSON, KeyValueHash, String, Time, UUID) have the following operators:

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| IS [NOT] NULL | isNull: `Boolean` | Is null (if `true` is specified) or is not null (if `false` is specified). |
| = | equalTo: `T` | Equal to the specified value. |
| <> | notEqualTo: `T` | Not equal to the specified value. |
| IS DISTINCT FROM | distinctFrom: `T` | Not equal to the specified value, treating null like an ordinary value. |
| IS NOT DISTINCT FROM | notDistinctFrom: `T` | Equal to the specified value, treating null like an ordinary value. |
| IN (...) | in: `[T]` | Included in the specified list. |
| NOT IN (...) | notIn: `[T]` | Not included in the specified list. |
| < | lessThan: `T` | Less than the specified value. |
| <= | lessThanOrEqualTo: `T`| Less than or equal to the specified value. |
| > | greaterThan: `T` | Greater than the specified value. |
| >= | greaterThanOrEqualTo: `T` | Greater than or equal to the specified value. |

where `T` is the type of the field being filtered.

The only exception is KeyValueHash (`hstore`) fields, for which no sort operators (<, <=, >, >=) are available.

The following types have additional operators:

#### InternetAddress (`inet`)

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| >> | contains: `InternetAddress` | Contains the specified internet address. |
| >>= | containsOrEqualTo: `InternetAddress` | Contains or equal to the specified internet address. |
| << | containedBy: `InternetAddress` | Contained by the specified internet address. |
| <<= | containedByOrEqualTo: `InternetAddress` | Contained by or equal to the specified internet address. |
| && | containsOrContainedBy: `InternetAddress` | Contains or contained by the specified internet address. |

#### JSON (`jsonb`)

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| @> | contains: `JSON` | Contains the specified JSON. |
| ? | containsKey: `String` | Contains the specified key. |
| ?& | containsAllKeys `[String]` | Contains all of the specified keys. |
| ?\| | containsAnyKeys: `[String]` | Contains any of the specified keys. |
| <@ | containedBy: `JSON` | Contained by the specified JSON. |

#### KeyValueHash (`hstore`)

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| @> | contains: `KeyValueHash` | Contains the specified KeyValueHash. |
| ? | containsKey: `String` | Contains the specified key. |
| ?& | containsAllKeys `[String]` | Contains all of the specified keys. |
| ?\| | containsAnyKeys: `[String]` | Contains any of the specified keys. |
| <@ | containedBy: `KeyValueHash` | Contained by the specified KeyValueHash. |

#### String

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| LIKE '%...%' | includes: `String` | Contains the specified string (case-sensitive). |
| NOT LIKE '%...%' | notIncludes: `String` | Does not contain the specified string (case-sensitive). |
| ILIKE '%...%' | includesInsensitive: `String` | Contains the specified string (case-insensitive). |
| NOT ILIKE '%...%' | notIncludesInsensitive: `String` | Does not contain the specified string (case-insensitive). |
| LIKE '...%' | startsWith: `String` | Starts with the specified string (case-sensitive). |
| NOT LIKE '...%' | notStartsWith: `String` | Does not start with the specified string (case-sensitive). |
| ILIKE '...%' | startsWithInsensitive: `String` | Starts with the specified string (case-insensitive). |
| NOT ILIKE '...%' | notStartsWithInsensitive: `String` | Does not start with the specified string (case-insensitive). |
| LIKE '%...' | endsWith: `String` | Ends with the specified string (case-sensitive). |
| NOT LIKE '%...' | notEndsWith: `String` | Does not end with the specified string (case-sensitive). |
| ILIKE '%...' | endsWithInsensitive: `String` | Ends with the specified string (case-insensitive). |
| NOT ILIKE '%...' | notEndsWithInsensitive: `String` | Does not end with the specified string (case-insensitive). |
| LIKE '...' | like: `String` | Matches the specified pattern (case-sensitive). An underscore (_) matches any single character; a percent sign (%) matches any sequence of zero or more characters. |
| NOT LIKE '...' | notLike: `String` | Does not match the specified pattern (case-sensitive). An underscore (_) matches any single character; a percent sign (%) matches any sequence of zero or more characters. |
| ILIKE '...' | likeInsensitive: `String` | Matches the specified pattern (case-insensitive). An underscore (_) matches any single character; a percent sign (%) matches any sequence of zero or more characters. |
| NOT ILIKE '...' | notLikeInsensitive: `String` | Does not match the specified pattern (case-insensitive). An underscore (_) matches any single character; a percent sign (%) matches any sequence of zero or more characters. |

The following operators are also available, but the mapping to SQL is more complex:

| GraphQL | Description |
| --- | --- |
| distinctFromInsensitive | Not equal to the specified value, treating null like an ordinary value (case-insensitive). |
| equalToInsensitive | Equal to the specified value (case-insensitive). |
| greaterThanInsensitive | Greater than the specified value (case-insensitive). |
| greaterThanOrEqualToInsensitive | Greater than or equal to the specified value (case-insensitive). |
| inInsensitive | Included in the specified list (case-insensitive). |
| lessThanInsensitive | Less than the specified value (case-insensitive). |
| lessThanOrEqualToInsensitive | Less than or equal to the specified value (case-insensitive). |
| notDistinctFromInsensitive | Equal to the specified value, treating null like an ordinary value (case-insensitive). |
| notEqualToInsensitive | Not equal to the specified value (case-insensitive). |
| notInInsensitive | Not included in the specified list (case-insensitive). |

The compiled SQL depends on the underlying PostgreSQL column type. Using case-insensitive operators with `text`/`varchar`/`char` columns will result in calling `lower()` on the operands. Using case-sensitive operators with `citext` columns will result in casting the operands to `text`.

For example, here is how the `equalTo`/`equalToInsensitive` operators compile to SQL:

| GraphQL operator   | PostgreSQL column type  | Compiled SQL               |
| ------------------ | ----------------------- | -------------------------- |
| equalTo            | `text`/`varchar`/`char` | `<col> = $1`               |
| equalTo            | `citext`                | `<col>::text = $1::text`   |
| equalToInsensitive | `text`/`varchar`/`char` | `lower(<col>) = lower($1)` |
| equalToInsensitive | `citext`                | `<col> = $1`               |

### Domains

Domain fields have the same operators as the domain's base type. For example, a domain type declared with `create domain ... as text check (...);` would have all of the String operators.

### Enums

Enum fields have the same operators as scalar fields.

### Ranges

Range fields have the same operators as scalar fields, plus the following range operators:

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| @> | contains: `T` | Contains the specified range. |
| @> | containsElement: `E` | Contains the specified value. |
| <@ | containedBy: `T` | Contained by the specified range. |
| && | overlaps `T` | Overlaps the specified range. |
| << | strictlyLeftOf: `T` | Strictly left of the specified range. |
| >> | strictlyRightOf: `T` | Strictly right of the specified range. |
| &< | notExtendsRightOf: `T` | Does not extend right of the specified range. |
| &> | notExtendsLeftOf: `T` | Does not extend left of the specified range. |
| -\|- | adjacentTo: `T` | Adjacent to the specified range. |

where `T` is the type of the range field being filtered and `E` is the element type of the range.

### Arrays

Array fields have the following operators:

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| IS [NOT] NULL | isNull: `Boolean` | Is null (if `true` is specified) or is not null (if `false` is specified). |
| = | equalTo: `[T]` | Equal to the specified value. |
| <> | notEqualTo: `[T]` | Not equal to the specified value. |
| IS DISTINCT FROM | distinctFrom: `[T]` | Not equal to the specified value, treating null like an ordinary value. |
| IS NOT DISTINCT FROM | notDistinctFrom: `[T]` | Equal to the specified value, treating null like an ordinary value. |
| < | lessThan: `[T]` | Less than the specified value. |
| <= | lessThanOrEqualTo: `[T]`| Less than or equal to the specified value. |
| > | greaterThan: `[T]` | Greater than the specified value. |
| >= | greaterThanOrEqualTo: `[T]` | Greater than or equal to the specified value. |
| @> | contains: `[T]` | Contains the specified list of values. |
| <@ | containedBy: `[T]` | Contained by the specified list of values. |
| && | overlaps: `[T]` | Overlaps the specified list of values. |
| = ANY(...) | anyEqualTo: `T` | Any array item is equal to the specified value. |
| <> ANY(...) | anyNotEqualTo: `T` | Any array item is not equal to the specified value. |
| > ANY(...) | anyLessThan: `T` | Any array item is less than the specified value. |
| >= ANY(...) | anyLessThanOrEqualTo: `T` | Any array item is less than or equal to the specified value. |
| < ANY(...) | anyGreaterThan: `T` | Any array item is greater than the specified value. |
| <= ANY(...) | anyGreaterThanOrEqualTo: `T` | Any array item is greater than or equal to the specified value. |

where `T` is the item type of the array field being filtered.

### Logic

Complex logic can be expressed using the following logical operators:

| PostgreSQL | GraphQL | Description |
| --- | --- | --- |
| AND | and: `[T]` | Checks for all expressions in this list. |
| OR | or: `[T]` | Checks for any expressions in this list. |
| NOT | not: `T` | Negates the expression. |

## Examples

<details>

<summary>Null values</summary>

```graphql
query {
  allPosts(filter: {
    body: { isNull: true }
  }) {
    ...
  }
}
```

</details>

<details>

<summary>Non-null values</summary>

```graphql
query {
  allPosts(filter: {
    body: { isNull: false }
  }) {
    ...
  }
}
```

</details>

<details>

<summary>Comparison operator with scalar input</summary>

```graphql
query {
  allPosts(filter: {
    createdAt: { greaterThan: "2016-01-01" }
  }) {
    ...
  }
}
```

</details>

<details>

<summary>Comparison operator with array input</summary>

```graphql
query {
  allPosts(filter: {
    authorId: { in: [1, 2] }
  }) {
    ...
  }
}
```

</details>

<details>

<summary>Multiple comparison operators</summary>

Note: Objects with multiple keys are interpreted with an implicit `AND` between the conditions.

```graphql
query {
  allPosts(filter: {
    body: { isNull: false },
    createdAt: { greaterThan: "2016-01-01" }
  }) {
    ...
  }
}
```

</details>

<details>

<summary>Logical operator</summary>

```graphql
query {
  allPosts(filter: {
    or: [
      { authorId: { equalTo: 6 } },
      { createdAt: { greaterThan: "2016-01-01" } }
    ]
  }) {
    ...
  }
}
```

</details>

<details>

<summary>Compound logic</summary>

```graphql
query {
  allPosts(filter: {
    not: {
      or: [
        { authorId: { equalTo: 6 } },
        { createdAt: { greaterThan: "2016-01-01" } }
      ]
    }
  }) {
    ...
  }
}
```

</details>

<details>

<summary>Relations: Nested</summary>

```graphql
query {
  allPeople(filter: {
    firstName: { startsWith:"John" }
  }) {
    nodes {
      firstName
      lastName
      postsByAuthorId(filter: {
        createdAt: { greaterThan: "2016-01-01" }
      }) {
        nodes {
          ...
        }
      }
    }
  }
}
```

</details>

<details>

<summary>Relations: Root-level, many-to-one</summary>

> Requires `connectionFilterRelations: true`

```graphql
query {
  allPosts(filter: {
    personByAuthorId: { createdAt: { greaterThan: "2018-01-01" } }
  }) {
    ...
  }
}
```

A node passes the filter if a related node exists *and* the filter criteria for the related node are satisfied. (If a related node does not exist, the check fails.)

The `*Exists` Boolean field can be used to filter on the existence of a related node:

```graphql
query {
  allPosts(filter: {
    personByAuthorIdExists: true
  }) {
    nodes {
      id
    }
  }
}
```

The `*Exists` Boolean field is only exposed on nullable relations. For example, if the `post.author_id` column is defined as `not null`, a related `person` always exists, so the `personByAuthorIdExists` field is not exposed.

</details>

<details>

<summary>Relations: Root-level, one-to-one</summary>

> Requires `connectionFilterRelations: true`

```graphql
query {
  allPeople(filter: {
    accountByAccountId: { status: { equalTo: ACTIVE } }
  }) {
    ...
  }
}
```

A node passes the filter if a related node exists *and* the filter criteria for the related node are satisfied. (If a related node does not exist, the check fails.)

The `*Exists` Boolean field can be used to filter on the existence of a related node:

```graphql
query {
  allPeople(filter: {
    accountByAccountId: true
  }) {
    nodes {
      id
    }
  }
}
```

The `*Exists` Boolean field is only exposed on nullable relations. For example, if the `person.account_id` column is defined as `not null`, a related `account` always exists, so the `accountByAccountIdExists` field is not exposed.

</details>

<details>

<summary>Relations: Root-level, one-to-many</summary>

> Requires `connectionFilterRelations: true`

One-to-many relation fields require the filter criteria to be nested under `every`, `some`, or `none`.

```graphql
query {
  allPeople(filter: {
    postsByAuthorId: {
      some: {
        status: { equalTo: PUBLISHED }
      }
    }
  }) {
    nodes {
      id
    }
  }
}
```

The `*Exist` Boolean field can be used to filter on the existence of related records:

```graphql
query {
  allPeople(filter: {
    postsByAuthorIdExist: true
  }) {
    nodes {
      id
    }
  }
}
```

</details>

For additional examples, see the [tests](https://github.com/graphile-contrib/postgraphile-plugin-connection-filter/blob/master/__tests__/fixtures/queries/).

## Plugin Options

When using PostGraphile as a library, the following plugin options can be passed via `graphileBuildOptions`:

<details>

<summary>connectionFilterAllowedOperators</summary>

Restrict filtering to specific operators:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterAllowedOperators: [
      "isNull",
      "equalTo",
      "notEqualTo",
      "distinctFrom",
      "notDistinctFrom",
      "lessThan",
      "lessThanOrEqualTo",
      "greaterThan",
      "greaterThanOrEqualTo",
      "in",
      "notIn",
    ],
  },
})
```

</details>

<details>

<summary>connectionFilterAllowedFieldTypes</summary>

Restrict filtering to specific field types:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterAllowedFieldTypes: ["String", "Int"],
  },
})
```

The available field types will depend on your database schema.

</details>

<details>

<summary>connectionFilterArrays</summary>

Enable/disable filtering on PostgreSQL arrays:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterArrays: false, // default: true
  },
})
```

</details>

<details>

<summary>connectionFilterComputedColumns</summary>

Enable/disable filtering by computed columns:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterComputedColumns: false, // default: true
  },
})
```

Consider setting this to `false` and using `@filterable` [smart comments](https://www.graphile.org/postgraphile/smart-comments/) to selectively enable filtering:

```sql
create function app_public.foo_computed(foo app_public.foo)
  returns ... as $$ ... $$ language sql stable;

comment on function app_public.foo_computed(foo app_public.foo) is E'@filterable';
```

</details>

<details>

<summary>connectionFilterOperatorNames</summary>

Use alternative names (e.g. `eq`, `ne`) for operators:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterOperatorNames: {
      equalTo: "eq",
      notEqualTo: "ne",
    },
  },
})
```

</details>

<details>

<summary>connectionFilterRelations</summary>

Enable/disable filtering on related fields:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterRelations: true, // default: false
  },
})
```

</details>

<details>

<summary>connectionFilterSetofFunctions</summary>

Enable/disable filtering on functions that return `setof`:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterSetofFunctions: false, // default: true
  },
})
```

Consider setting this to `false` and using `@filterable` [smart comments](https://www.graphile.org/postgraphile/smart-comments/) to selectively enable filtering:

```sql
create function app_public.some_foos()
  returns setof ... as $$ ... $$ language sql stable;

comment on function app_public.some_foos() is E'@filterable';
```

</details>

<details>

<summary>connectionFilterLogicalOperators</summary>

Enable/disable filtering with logical operators (`and`/`or`/`not`):

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterLogicalOperators: false, // default: true
  },
})
```

</details>

<details>

<summary>connectionFilterAllowNullInput</summary>

Allow/forbid `null` literals in input:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterAllowNullInput: true, // default: false
  },
})
```

When `false`, passing `null` as a field value will throw an error.
When `true`, passing `null` as a field value is equivalent to omitting the field.

</details>

<details>

<summary>connectionFilterAllowEmptyObjectInput</summary>

Allow/forbid empty objects (`{}`) in input:

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    connectionFilterAllowEmptyObjectInput: true, // default: false
  },
})
```

When `false`, passing `{}` as a field value will throw an error.
When `true`, passing `{}` as a field value is equivalent to omitting the field.

</details>

## Development

To establish a test environment, create an empty PostgreSQL database with C collation (required for consistent ordering of strings) and set a `TEST_DATABASE_URL` environment variable with your database connection string.

```bash
createdb graphile_test_c --template template0 --lc-collate C
export TEST_DATABASE_URL=postgres://localhost:5432/graphile_test_c
yarn
yarn test
```
