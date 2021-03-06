### Custom Query Input Types for GraphQL Server

We saw how the paging-control query input `...connectionArgs` are defined in the standard `graphql-relay` package. Now let's take a look at the custom Input Types that are set up to provide a uniform way of query arguments formation across our implementation of GraphQL API - `src/relay-queries/input-types-get-query.js`.

### Sort Order Arguments

There is no standard on defining the output's Sort Order in GraphQL queries, however it is a good idea to adopt a format used in some popular [big-name APIs](https://developer.github.com):

```
export const OrderByType = new GraphQLList(
  new GraphQLInputObjectType({
    name: 'OrderBy',
    fields: {
      field: { type: GraphQLString },
      direction: {
        type: OrderByDirectionType,
        defaultValue: OrderByDirectionType.DESC
      }
    }
  })
);

export const OrderByDirectionType = new GraphQLEnumType({
  name: 'OrderByDirection',
  values: {
    DESC: { value: -1 },
    ASC: { value: 1 }
  }
});
```

In this definition, the sorting is called *Order By* and it consists of two parameters: `field` and `direction`. The latter is limited to two values to be used on the client side: `DESC` (default) and `ASC`. On the server side, those come in as `-1` or `1`.

Notice the use of GraphQL standard types, like `GraphQLEnumType` - we've seen similar types across schema definitions we reviewed earlier. So, if you need to define your type, you build it from one or more standard ones that come with the `graphql` package.

### Note on GraphQLDateTime

Somehow, Date/Time has not been included into the `graphql` package, so we use a 3rd party one:

```
import { GraphQLDateTime } from 'graphql-iso-date';
```

GraphQL documentation talks about [Scalar types](https://graphql.org/learn/schema/#scalar-types), which are `Int`, `Float`, `String`, `Boolean` and `ID`, and mentions that a *Date* type can be created if needed. Oh, well, it *is* needed developing any worthwhile business API, and yet it's not in the `graphql` package. From Issues entered for the package, it appears that the authors don't feel like there is a strong valuable standard to base the Date implementation on for JS. Thankfully, `graphql-iso-date` provides exactly what we need, never mind another external dependency plugged in - it's part of the JS development experience.

### Query Resolver Arguments

This part of `listUsers` query definition enables passing a generic `name-value` list from the client:

```
    resolverArgs: {
      type: inputTypes.QueryResolverArgsType
    },
```

`QueryResolverArgsType` custom type is defined as a `GraphQLList`:

```
export const QueryResolverArgsType = new GraphQLList(
  new GraphQLInputObjectType({
    name: 'QueryResolverArgs',
    fields: {
      param: { type: GraphQLString },
      value: { type: GraphQLString }
    }
  })
);
```

In a client query, the syntax for passing `resolverArgs` would look like this:

```
sampleQuery(<some args>, 
  resolverArgs: [{param: "name", value: "John"}, {param: "email", value: "jp@example.com"}] ,
  <some more args>){
   <fields to return>
}
```

Not a particularly appealing syntax, but allows adjusting the query implementation by adding parameters to it without modifying the schema. Alternatively, our Input Type could have looked like this:

```
export const userQueryArgsType = 
  new GraphQLInputObjectType({
    name: 'UserQueryArgs',
    fields: {
      name: { type: GraphQLString },
      email: { type: GraphQLString }
    }
  })
;
```

which would evaluate to a more readable format on the query side:

```
sampleQuery(<some args>, 
  userArgs: {
       {name: "John"}, email: "jp@example.com"}
     } ,
  <some more args>){
   <fields to return>
}
```

Decide which style makes more sense for you. Keep in mind, the browser client users don't ever see GraphQL syntax, only the front end developers do. So, by making the schema more readable you assist your fellow developers, but require them to reload the client side schema file when you post updates.

### Input with Scalar Types

GraphQL does not enforce the use of Input Types as query arguments, although the documentation and some official samples recommend doing so. Fields of *Scalar* Types can be used in the `args` section of a query definition, e.g.:

```
export const getUserSampleQuery = {
  type: UserType,
  description: 'User',
  args: {
    name: {
      type: GraphQLString
    },
    email: {
      type: GraphQLString
    }
  },
  resolve: (obj, args, viewer, info) =>
    resolveGetUser(obj, args, viewer, info)
};
```

Then the client query will look like:

```
getUserSampleQuery(name: "John", email: "jp@example.com")
}
   <fields to return>
}
```

### filterValues Argument

This argument is a string:

```
export const FilterValuesType = new GraphQLInputObjectType({
  name: 'FilterValues',
  fields: {
    filterValuesString: { type: GraphQLString }
  }
});
```

As we saw earlier, the generic paging procedure passes this argument directly into the database query function where it can be picked up by the MongoDB query builder, if coded so by the developer. This looks like a powerful way to allow constructing free-form DB queries on the client, using MongoDB syntax, if you share the MongoDB schemas with the front end developers, of course. In the general web development practice, passing unedited strings into the DB engine is *absolutely forbidden* as it is a *huge security risk*. If you're building a fully-controlled communication flow with some trusted and intelligent GraphQL clients - sure, you may choose to code parts of your database query logic on the client in the actual MongoDB syntax and pass it to the server via this argument, e.g.:

```
getOrdersSampleQuery( filterValuesString: "{\"order_items.item_id\": \"1HEOx6FnC7cM\"}")
}
   <fields to return>
}
```

### GraphQL Client Query Writing Recap

So, we've looked at the arguments and, along the way, reviewed how to form the required return fields portion of a client GraphQL query. You can try removing individual fields from the `listUsers` sample you run in GraphiQL and execute the query again - those fields will not be included into the returned dataset in the right pane. As simple as that. Also, if you add fields that don't exist in the schema - GraphiQL will complain and won't allow launching the query.
<br>
Next, let's trace how the GraphQL query is actually processed by the server engine