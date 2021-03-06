# Omanyd

A simple and experimental dynamodb data mapper.

[![Coverage Status](https://coveralls.io/repos/github/tgandrews/omanyd/badge.svg?branch=main)](https://coveralls.io/github/tgandrews/omanyd?branch=main)

## Features

- Simplified data modeling and mapping to DynamoDB types
- Data validation using [Joi](https://joi.dev/)
- [Autogenerating IDs](#Creating)
- Complete typescript typings
- Basic global indexes
- Range key
- Lists

### Missing features

- Parallel scans
- Paging
- Complex querying
- Number and binary sets
- Date types
- Local indexes

## Installation

npm: `npm install omanyd joi`  
yarn: `yarn add omanyd joi`

Both packages come with the necessary types so no need to download
anything additional for typescript.

## Getting Started

Set the AWS environment variables before running the program

```bash
AWS_REGION="REGION" \
AWS_ACCESS_KEY_ID="ACCESS KEY ID" \
AWS_SECRET_ACCESS_KEY="SECRET ACCESS KEY" \
node app.js
```

These will already by defined for you if you are running in ec2 or lambda.

To host on services like [vercel](https://vercel.com) you need to specify your own AWS environment
variables but cannot set them using the standard names above. To do this you can specify them using
an `OMANYD_` prefix so they become:

```bash
OMANYD_AWS_REGION="REGION"
OMANYD_AWS_ACCESS_KEY_ID="ACCESS KEY ID"
OMANYD_AWS_SECRET_ACCESS_KEY="SECRET ACCESS KEY"
```

For running locally we recommend using the [official dynamodb docker container](https://hub.docker.com/r/amazon/dynamodb-local)
and then providing an additional environment variable to override the dynamodb url.

```bash
DYNAMODB_URL=http://localhost:8000
```

### Define a Store

Stores are defined through `define`. You provide the table name, schema and hashKey definition.
Stores are the accessors to underlying dymamodb table.

```ts
import Omanyd from "omanyd";
import Joi from "joi";

interface Tweet {
  id: string;
  content: string;
}
const TweetStore = Omanyd.define<Tweet>({
  name: "Tweet",
  hashKey: "id",
  schema: Joi.object({
    id: Omanyd.types.id(),
    content: Joi.string(),
  }),
});
```

### Create tables (for testing)

You can create tables for use locally during tests but should be managing this with a proper tool
[IaC](https://en.wikipedia.org/wiki/Infrastructure_as_code) for production.

This expects all stores defined using the define method above before use. It will skip creating a
table if it already exists so this cannot be used for modifying a table definition.

```js
import { createTables } from "omanyd";

await createTables();
```

### Delete Tables (for testing)

You can delete tables for use locally during tests but should be managing this with a proper tool
[IaC](https://en.wikipedia.org/wiki/Infrastructure_as_code) for production.

This expects all stores defined using the define method above before use. It then clears all saved
definitions so they can be redefined.

```ts
import { deleteTables } from "omanyd";

await deleteTables();
```

### Clear Tables (for testing)

You can clear all existing data from known tables by deleting and then
redefining the tables. This is a quick function for doing that for you.

```ts
import { clearTables } from "omanyd";

await clearTables();
```

### Creating

Once you have defined your store you can create models from it and unless you provide an id then one
will be created for you automatically as the `omanyd.types.id()` was used in the [definition above](#Define%20a%20model)

```ts
const tweet = await TweetStore.create({ content: "My first tweet" });
console.log(tweet);
/*
 * { id: "958f2b51-774a-436a-951e-9834de3fe559", content: "My first tweet"  }
 */
```

### Reading one - getting by hash key

Now that we have some data in the store we can now read it. The quickest way is reading directly by the hash key.

```ts
const readTweet = await TweetStore.getByHashKey(
  "958f2b51-774a-436a-951e-9834de3fe559"
);
console.log(readTweet);
/*
 * { id: "958f2b51-774a-436a-951e-9834de3fe559", content: "My first tweet"  }
 */
```

### Reading many - scanning

If we want all of the items in the store we can use a scan. DynamoDB scans come with some [interesting caveats](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Scan.html).

```ts
await Promise.all([
  TweetStore.create({ content: "My second tweet" }),
  TweetStore.create({ content: "My third tweet" }),
  TweetStore.create({ content: "My fourth tweet" }),
]);

const tweets = await TweetStore.scan();

console.log(tweets);
/* [
 *   { id: "958f2b51-774a-436a-951e-9834de3fe559", content: "My first tweet"  },
 *   { id: "aa6ea347-e3d3-4c73-8960-709fa47e3a4c", content: "My second tweet"  },
 *   { id: "9cd6b18a-eafd-49c2-8f0f-d3bf8e75c26e", content: "My third tweet"  },
 *   { id: "fc446fcd-d65a-4ae2-ba9f-6bd94aae8705", content: "My fourth tweet"  }
 * ]
 */
```

### Updating an item - putting

Now that we have saved and read an item lets update it with a new value.

```ts
const updatedTweet = await TweetStore.put({
  ...tweet,
  content: "I hope you are having a good day",
});

console.log(updatedTweet);
/*
 * { id: "958f2b51-774a-436a-951e-9834de3fe559", content: "I hope you are having a good day"  }
 */
```

### Deleting an item

Now lets get rid of what we have created.

```ts
await TweetStore.deleteByHashKey("958f2b51-774a-436a-951e-9834de3fe559");
const readTweet = await TweetStore.getByHashKey(
  "958f2b51-774a-436a-951e-9834de3fe559"
);
console.log(readTweet);
/*
 * null
 */
```

## Advanced Features

### Range keys

It is possible define a composite key for a model so you can have a repeating hash key. This is
great for features like versioning. To do this you need to define this range key as part of the
definition and then you have access to `getByHashAndRangeKey`.

```ts
import Omanyd from "omanyd";
import Joi from "joi";

interface Document {
  id: string;
  version: string;
  content: string;
}
const DocumentStore = Omanyd.define<User>({
  name: "Documents",
  hashKey: "id",
  schema: Joi.object({
    id: Omanyd.types.id(),
    version: Joi.string().required(),
    email: Joi.string().required(),
  }),
});

// Assuming table has been created separately
const original = await DocumentStore.create({
  id,
  version: "1.0",
  content: "hello",
});
await DocumentStore.create({
  id: original.id,
  version: "2.0",
  content: "hello world",
});

const document = await DocumentStore.getByHashAndRangeKey(id, "2.0");
console.log(document);
/*
 * { id: "e148f2ca-e86d-4c5b-8826-2dbb101a3553", content: "hello world", version: "2.0"  }
 */
```

### Global indexes

It is possible to quickly access documents by keys other than their hash key. This is done through
indexes.

Indexes should be created as a part of your table creation but need to be defined with Omanyd so
they can be used at run time correctly.

```ts
import Omanyd from "omanyd";
import Joi from "joi";

interface User {
  id: string;
  content: string;
}
const UserStore = Omanyd.define<User>({
  name: "Users",
  hashKey: "id",
  schema: Joi.object({
    id: Omanyd.types.id(),
    email: Joi.string().required(),
  }),
  indexes: [
    {
      name: "EmailIndex",
      type: "global",
      hashKey: "email",
    },
  ],
});

// Assuming table and index have been created separately

await UserStore.create({ email: "hello@world.com" });

const user = await UserStore.getByIndex("EmailIndex", "hello@world.com");
console.log(user);
/*
 * { id: "958f2b51-774a-436a-951e-9834de3fe559", email: "hello@world.com"  }
 */
```

## History

Omanyd was originally inspired by [dynamodb](https://www.npmjs.com/package/dynamodb) and [dynamoose](https://www.npmjs.com/package/dynamoose)

## Support

Omanyd is provided as-is, free of charge. For support, you have a few choices:

- Ask your support question on [Stackoverflow.com](http://stackoverflow.com), and tag your question with **omanyd**.
- If you believe you have found a bug in omanyd, please submit a support ticket on the [Github Issues page for omanyd](http://github.com/tgandrews/omanyd/issues).
