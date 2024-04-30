# kysely-libsql

A [Kysely][kysely] dialect for [libSQL/sqld][sqld], using the [Hrana][hrana-client-ts] protocol over a WebSocket.

[kysely]: https://github.com/koskimas/kysely
[sqld]: https://github.com/libsql/sqld
[hrana-client-ts]: https://github.com/libsql/hrana-client-ts

## Why and how
This is a fork of `@libsql/kysely-libsql` and is exactly the same, except for 1 thing:
It passes the `columnDecltypes` property into the results back to Kysely.

SQLite does not support booleans. It uses a `1` or `0` to encode booleans.
When sending data (creates/updates) to libSQL, it converts booleans to a `1` or `0`.
But when querying data, it does not.

So, we have to do this on our own in our application code. A good way to do this is through
a Kysely plugin, so that it applies to all our queries automatically.

In order to write such a plugin, we'll need the column information. LibSQL passes that information
using the `columnDecltypes` property, but it is not passed through by the @libsql/kysely-libsql
driver (the Kysely types don't allow it).

This fork passes it through, and casts it to avoid type errors.
The plugin in your application can then use it, but it needs to expand the Kysely types.

## Installation

```shell
npm install @voinik/kysely-libsql
```

## Using the exposed `columnDecltypes` in your app
Here's an example:

A nice way to organize custom types is to have a folder in your app root called
"customTypings" for example. Tell your tsconfig about it by adding this to it:
```ts
    "include": [
        // other includes here...
        "./customTypings/**/*.ts"
    ],
```

Then, add a `kysely.d.ts` file in that folder with these contents:
```ts
// ./customTypings/kysely.d.ts
export {}

declare module 'kysely' {
    interface QueryResult {
        // This is coming from the custom kysely driver (@voinik/kysely-libsql).
        // It is the same as @libsql/kysely-libsql, except that it adds the columnDecltypes
        // property, which tells us what column types are in the results. This allows us to
        // cast incoming boolean integers (sqlite doesn't have booleans) as actual booleans
        // at runtime.
        columnDecltypes: Array<(string | undefined)>;
    }
}
```
Next, you can create a Kysely plugin with your new types:
```ts
// src/server/db/plugins/sqliteBooleanPlugin.ts
import { type KyselyPlugin } from 'kysely';

export const sqliteBooleanPlugin: KyselyPlugin = {
    transformQuery: args => args.node,
    transformResult: async args => {
        const indices = [];
        for (let i = 0; i < args.result.columnDecltypes.length; i++) {
            if (args.result.columnDecltypes[i] === 'boolean') {
                indices.push(i);
            }
        }

        if (indices.length === 0) {
            return args.result;
        }

        for (const row of args.result.rows) {
            const keys = Object.keys(row);
            for (const index of indices) {
                row[keys[index]!] = Boolean(row[keys[index]!]);
            }
        }

        return args.result;
    },
};
```

And finally, create your Kysely instance with the plugin:
```ts
// src/server/db/client.ts
import { LibsqlDialect } from '@voinik/kysely-libsql';
import { Kysely } from 'kysely';

import { sqliteBooleanPlugin } from './plugins/sqliteBooleanPlugin';
import { type Database } from './tables/Database';

export const db = new Kysely<Database>({
    dialect: new LibsqlDialect({
        url: process.env.DB_URL, // Replace with your own env var
        authToken: process.env.DB_TOKEN, // Replace with your own env var if you have it
    }),
    plugins: [sqliteBooleanPlugin],
});
```
**Make sure to replace the import paths with whatever your setup is!**

Now, all queries that select a boolean column will actually give you back
`true` or `false` instead of `1` or `0`.

## Basic usage

Pass a `LibsqlDialect` instance as the `dialect` when creating the `Kysely` object:

```typescript
import { Kysely } from "kysely";
import { LibsqlDialect } from "@voinik/kysely-libsql";

interface Database {
    ...
}

const db = new Kysely<Database>({
    dialect: new LibsqlDialect({
        url: "libsql://localhost:8080?tls=0",
        authToken: "<token>", // optional
    }),
});
```

Instead of a `url`, you can also pass an instance of `Client` from [`@libsql/hrana-client`][hrana-client-ts] as `client`:

```typescript
import * as hrana from "@libsql/hrana-client";
// Alternatively, the `kysely-libsql` package reexports the `hrana-client`
//import { hrana } from "@voinik/kysely-libsql";

const client = hrana.open("ws://localhost:2023");

const db = new Kysely<Database>({
    dialect: new LibsqlDialect({ client }),
});

// after you are done with the `db`, you must close the `client`:
client.close();
```

## Supported URLs

The library accepts the [same URL schemas][supported-urls] as [`@libsql/client`][libsql-client-ts] except `file:`:

- `http://` and `https://` connect to a libsql server over HTTP,
- `ws://` and `wss://` connect to the server over WebSockets,
- `libsql://` connects to the server using the default protocol (which is now HTTP). `libsql://` URLs use TLS by default, but you can use `?tls=0` to disable TLS (e.g. when you run your own instance of the server locally).

Connecting to a local SQLite file using `file:` URL is not supported; we suggest that you use the native Kysely dialect for SQLite.

[libsql-client-ts]: https://github.com/libsql/libsql-client-ts
[supported-urls]: https://github.com/libsql/libsql-client-ts#supported-urls

## License

This project is licensed under the MIT license.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in `@voinik/kysely-libsql` by you, shall be licensed as MIT, without any additional terms or conditions.
