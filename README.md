# GraphQL Batch Resolver

An alternative mechanism for batching the resolution of GraphQL fields.

```js
import { GraphQLObjectType, GraphQLString } from 'graphql';
import { createBatchResolver } from 'graphql-batch-resolve';

const UserType = new GraphQLObjectType({
  // ...
});

const QueryType = new GraphQLObjectType({
  name: 'Query',
  fields: {
    user: {
      type: UserType,
      resolve: createBatchResolver(async (sources, args, context) => {
        const { db } = context;
        const users = await db.loadUsersByIds(sources.map(({ id }) => id));
        return users;
      }),
    },
  },
});
```

## Installation

`graphql-batch-resolve` has a peer dependency on `graphql`, so make sure you have installed that package as well.

```
npm install --save graphql graphql-batch-resolve
```

[`graphql`]: https://www.npmjs.com/package/graphql

## Why?

GraphQL is a powerful data querying language for both frontend and backend developers. However, because of how GraphQL queries are executed, it can be difficult to define an efficient GraphQL schema. Take for example the following query:

```graphql
{
  users(limit: 5) {
    name
    friends(limit: 5) {
      name
    }
  }
}
```

This demonstrates the power of GraphQL to select arbitrarily nested data. Yet it is a difficult pattern to optimize from the schema developer’s perspective. If we naïvely translate this GraphQL query into say, SQL, we get the following psudo queries:

```
Select the first 5 users.
Select the first 5 friends for the first user.
Select the first 5 friends for the second user.
Select the first 5 friends for the third user.
Select the first 5 friends for the fourth user.
Select the first 5 friends for the fifth user.
```

We have an N+1 problem! For every user we are executing a database query. This is noticably inefficient and does not scale. What happens when we have:

```graphql
{
  users(limit: 5) {
    name
    friends(limit: 5) {
      name
      friends(limit: 5) {
        name
        friends(limit: 5) {
          name
        }
      }
    }
  }
}
```

This turns into 155 queries!

The canonical solution to this problem is to use [`dataloader`][] which supposedly implements a pattern that Facebook uses to optimize their GraphQL API in JavaScript. `dataloader` is excellent for batching queries with a simple key. For example this query:

[`dataloader`]: https://github.com/facebook/dataloader

```graphql
{
  users(limit: 5) {
    name
    bestFriend {
      name
    }
  }
}
```

Is easy to optimize this GraphQL query with `dataloader` because assumedly the value we use to fetch the `bestFriend` is a scalar. A simple string identifier for instance. However, when we add arguments into the equation:

```graphql
{
  users(limit: 5) {
    name
    friends(limit: 5) {
      name
    }
    friends(limit: 5, offset: 5) {
      name
    }
  }
}
```

All of a sudden the keys are not simple scalars. If we wanted to use `dataloader` we might need to use *two* `dataloader` instances. One for `friends(limit: 5)` and one for `friends(limit: 5, offset: 5)` and then on each instance we can use a simple key. An implementation like this can get very complex very quickly and is likely not what you want to spend your time building.

This package offers an alternative to the `dataloader` batching strategy. This package implements an opinionated batching strategy customized for GraphQL. Instead of batching using a simple key, this package batches by the *GraphQL field*. So for example, let us again look at the following query:

```graphql
{
  users(limit: 5) {
    name
    friends(limit: 5) { # Batches 5 executions.
      name
      friends(limit: 5) { # Batches 25 executions.
        name
        friends(limit: 5) { # Batches 125 executions.
          name
        }
      }
    }
  }
}
```

Here we would only have *4* executions instead of 155. One for the root field, one for the first `friends` field, one for the second `friends` field, and so on. This is a powerful alternative to `dataloader` in a case where `dataloader` falls short.

## When do I use `dataloader` and when do I use `graphql-resolve-batch`?

If you answer yes to any of these questions:

- Do you have a simple primitive key like a string or number that you can use to batch with?
- Do you want to batch requests across your entire schema?
- Do you want to cache data with the same key so that it does not need to be re-requested?

Use `dataloader`. But for all of the cases where `dataloader` is useful, `graphql-resolve-batch` will likely also be useful. If you find `dataloader` to complex to set up, and its benefits not very attractive you could just use `graphql-resolve-batch` for everywhere you need to hit the database.

However, if you answer yes to any of these questions:

- Does your field have arguments?
- Is it hard for you to derive a primitive value from your source values for your field?
- Do you not have the ability to add any new values to `context`? (such as in an embedded GraphQL schema)

You almost certainly want to use `graphql-resolve-batch`. If you are using `dataloader` then `graphql-resolve-batch` will only be better in a few niche cases. However, `graphql-resolve-batch` is easier to set up.

## Credits

Enjoy efficient GraphQL APIs? Follow the author, [`@calebmer`][] on Twitter for more awesome work like this.

[`@calebmer`]: http://twitter.com/calebmer