---
title: Mocking
order: 306
description: Mock your GraphQL data based on a schema.
---

The strongly-typed nature of a GraphQL API lends itself extremely well to mocking. This is an important part of a GraphQL-First development process, because it enables frontend developers to build out UI components and features without having to wait for a backend implementation.

Even when the UI is already built, it can let you test your UI without waiting on slow database requests, or build out a component harness using a tool like React Storybook without needing to start a real GraphQL server.

## Default mock example

Let's take a look at how we can mock a GraphQL schema with just one line of code, using the default mocking logic you get out of the box with `graphql-tools`. To start, let's grab the schema definition string from the `makeExecutableSchema` example [in the "Generating a schema" article](/tools/graphql-tools/generate-schema.html#example).

```js
import { makeExecutableSchema, addMockFunctionsToSchema } from 'graphql-tools';
import { graphql } from 'graphql';

// Fill this in with the schema string
const schemaString = `...`;

// Make a GraphQL schema with no resolvers
const schema = makeExecutableSchema({ typeDefs: schemaString });

// Add mocks, modifies schema in place
addMockFunctionsToSchema({ schema });

const query = `
query tasksForUser {
  user(id: 6) { id, name }
}
`;

graphql(schema, query).then((result) => console.log('Got result', result));
```
 
> Note: If your schema has custom scalar types, you still need to define the `__serialize`, `__parseValue`, and `__parseLiteral` functions, and pass them inside the second argument to `makeExecutableSchema`.

## Customizing mocks

This mocking logic simply looks at your schema and makes sure to return a string where your schema has a string, a number for a number, etc. So you can already get the right shape of result. But if you want to use the mocks to do sophisticated testing, you will likely want to customize them to your particular data model.

This is where the second argument to `addMockFunctionsToSchema` comes in, which is an object that describes your desired mocking logic. This is similar to the `resolverMap` in `makeExecutableSchema`, but has a few extra features aimed at mocking.

You can specify a function that is called for a specific type in the schema, for example:

```js
{
  Int: () => 6,
  Float: () => 22.1,
  String: () => 'Hello',
}
```

You can also use this to describe object types, and the fields can be functions too:

```js
{
  Person: () => ({
    name: casual.name,
    age: () => casual.integer(0, 120),
  }),
}
```

In this example, we are using [casual](https://github.com/boo1ean/casual), a fake data generator for JavaScript, so that we can get a different result every time the field is called. You might want to use a collection of fake objects, or a generator that always uses a consistent seed, if you are planning to use the data for testing.

You can also use the MockList constructor to automate mocking a list:

```js
{
  Person: () => {
    // a list of length between 2 and 6
    friends: () => new MockList([2,6]),
    // a list of three lists of two items: [[1, 1], [2, 2], [3, 3]]
    listOfLists: () => new MockList(3, () => new MockList(2)),
  },
}
```

Since the mock functions on fields are actually just GraphQL resolvers, you can use arguments and context in them as well:

```js
{
  Person: () => {
    // the number of friends in the list now depends on numPages
    paginatedFriends: (root, { numPages }) => new MockList(numPages * PAGE_SIZE),
  },
}
```

You can read some background and flavor on this approach in our blog post, ["Mocking your server with one line of code"](https://medium.com/apollo-stack/mocking-your-server-with-just-one-line-of-code-692feda6e9cd).

## API

### addMockFunctionsToSchema

```js
import { addMockFunctionsToSchema } from 'graphql-tools';

addMockFunctionsToSchema({
  schema,
  mocks: {},
  preserveResolvers: false,
});
```

Given an instance of GraphQLSchema and a mock object, `addMockFunctionsToSchema` modifies the schema in place to return mock data for any valid query that is sent to the server. If `mocks` is not passed, the defaults will be used for each of the scalar types. If `preserveResolvers` is set to `true`, existing resolve functions will not be overwritten to provide mock data. This can be used to mock some parts of the server and not others.

### MockList

```js
import { MockList } from 'graphql-tools';

new MockList(length: number | number[], mockFunction: Function);
```

This is an object you can return from your mock resolvers which calls the `mockFunction` once for each list item. The first argument can either be an exact length, or an inclusive range of possible lengths for the list, in case you want to see how your UI responds to varying lists of data.
