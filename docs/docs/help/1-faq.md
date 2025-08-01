# FAQ

## How do I update a database schema?

One of the main responsibilities of TypeORM is to keep your database tables in sync with your entities.
There are two ways that help you achieve this:

-   Use `synchronize: true` in data source options:

    ```typescript
    import { DataSource } from "typeorm"

    const myDataSource = new DataSource({
        // ...
        synchronize: true,
    })
    ```

    This option automatically syncs your database tables with the given entities each time you run this code.
    This option is perfect during development, but in production you may not want this option to be enabled.

-   Use command line tools and run schema sync manually in the command line:

    ```shell
    typeorm schema:sync
    ```

    This command will execute schema synchronization.

Schema sync is extremely fast.
If you are considering to disable synchronize option during development because of performance issues,
first check how fast it is.

## How do I change a column name in the database?

By default, column names are generated from property names.
You can simply change it by specifying a `name` column option:

```typescript
@Column({ name: "is_active" })
isActive: boolean;
```

## How can I set the default value to some function, for example `NOW()`?

`default` column option supports a function.
If you are passing a function which returns a string,
it will use that string as a default value without escaping it.
For example:

```typescript
@Column({ default: () => "NOW()" })
date: Date;
```

## How to do validation?

Validation is not part of TypeORM because validation is a separate process
not really related to what TypeORM does.
If you want to use validation use [class-validator](https://github.com/pleerock/class-validator) - it works perfectly with TypeORM.

## What does "owner side" in a relations mean or why we need to use `@JoinColumn` and `@JoinTable`?

Let's start with `one-to-one` relation.
Let's say we have two entities: `User` and `Photo`:

```typescript
@Entity()
export class User {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    name: string

    @OneToOne()
    photo: Photo
}
```

```typescript
@Entity()
export class Photo {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    url: string

    @OneToOne()
    user: User
}
```

This example does not have a `@JoinColumn` which is incorrect.
Why? Because to make a real relation, we need to create a column in the database.
We need to create a column `userId` in `photo` or `photoId` in `user`.
But which column should be created - `userId` or `photoId`?
TypeORM cannot decide for you.
To make a decision, you must use `@JoinColumn` on one of the sides.
If you put `@JoinColumn` in `Photo` then a column called `userId` will be created in the `photo` table.
If you put `@JoinColumn` in `User` then a column called `photoId` will be created in the `user` table.
The side with `@JoinColumn` will be called the "owner side of the relationship".
The other side of the relation, without `@JoinColumn`, is called the "inverse (non-owner) side of relationship".

It is the same in a `@ManyToMany` relation. You use `@JoinTable` to show the owner side of the relation.

In `@ManyToOne` or `@OneToMany` relations, `@JoinColumn` is not necessary because
both decorators are different, and the table where you put the `@ManyToOne` decorator will have the relational column.

`@JoinColumn` and `@JoinTable` decorators can also be used to specify additional
join column / junction table settings, like join column name or junction table name.

## How do I add extra columns into many-to-many (junction) table?

It's not possible to add extra columns into a table created by a many-to-many relation.
You'll need to create a separate entity and bind it using two many-to-one relations with the target entities
(the effect will be same as creating a many-to-many table),
and add extra columns in there. You can read more about this in [Many-to-Many relations](../relations/4-many-to-many-relations.md#many-to-many-relations-with-custom-properties).

## How to handle outDir TypeScript compiler option?

When you are using the `outDir` compiler option, don't forget to copy assets and resources your app is using into the output directory.
Otherwise, make sure to setup correct paths to those assets.

One important thing to know is that when you remove or move entities, the old entities are left untouched inside the output directory.
For example, you create a `Post` entity and rename it to `Blog`,
you no longer have `Post.ts` in your project. However, `Post.js` is left inside the output directory.
Now, when TypeORM reads entities from your output directory, it sees two entities - `Post` and `Blog`.
This may be a source of bugs.
That's why when you remove and move entities with `outDir` enabled, it's strongly recommended to remove your output directory and recompile the project again.

## How to use TypeORM with ts-node?

You can prevent compiling files each time using [ts-node](https://github.com/TypeStrong/ts-node).
If you are using ts-node, you can specify `ts` entities inside data source options:

```javascript
{
    entities: ["src/entity/*.ts"],
    subscribers: ["src/subscriber/*.ts"]
}
```

Also, if you are compiling js files into the same folder where your typescript files are,
make sure to use the `outDir` compiler option to prevent
[this issue](https://github.com/TypeStrong/ts-node/issues/432).

Also, if you want to use the ts-node CLI, you can execute TypeORM the following way:

```shell
npx typeorm-ts-node-commonjs schema:sync
```

For ESM projects use this instead:

```shell
npx typeorm-ts-node-esm schema:sync
```

## How to use Webpack for the backend?

Webpack produces warnings due to what it views as missing require statements -- require statements for all drivers supported by TypeORM. To suppress these warnings for unused drivers, you will need to edit your webpack config file.

```javascript
const FilterWarningsPlugin = require('webpack-filter-warnings-plugin');

module.exports = {
    ...
    plugins: [
        //ignore the drivers you don't want. This is the complete list of all drivers -- remove the suppressions for drivers you want to use.
        new FilterWarningsPlugin({
            exclude: [/mongodb/, /mssql/, /mysql/, /mysql2/, /oracledb/, /pg/, /pg-native/, /pg-query-stream/, /react-native-sqlite-storage/, /redis/, /sqlite3/, /sql.js/, /typeorm-aurora-data-api-driver/]
        })
    ]
};
```

### Bundling Migration Files

By default Webpack tries to bundle everything into one file. This can be problematic when your project has migration files which are meant to be executed after bundled code is deployed to production. To make sure all your migrations can be recognized and executed by TypeORM, you may need to use "Object Syntax" for the `entry` configuration for the migration files only.

```javascript
const glob = require("glob")
const path = require("path")

module.exports = {
    // ... your webpack configurations here...
    // Dynamically generate a `{ [name]: sourceFileName }` map for the `entry` option
    // change `src/db/migrations` to the relative path to your migration folder
    entry: glob
        .sync(path.resolve("src/db/migrations/*.ts"))
        .reduce((entries, filename) => {
            const migrationName = path.basename(filename, ".ts")
            return Object.assign({}, entries, {
                [migrationName]: filename,
            })
        }, {}),
    resolve: {
        // assuming all your migration files are written in TypeScript
        extensions: [".ts"],
    },
    output: {
        // change `path` to where you want to put transpiled migration files.
        path: __dirname + "/dist/db/migrations",
        // this is important - we want UMD (Universal Module Definition) for migration files.
        libraryTarget: "umd",
        filename: "[name].js",
    },
}
```

Also, since Webpack 4, when using `mode: 'production'`, files are optimized by default which includes mangling your code in order to minimize file sizes. This breaks the migrations because TypeORM relies on their names to determine which has already been executed. You may disable minimization completely by adding:

```javascript
module.exports = {
    // ... other Webpack configurations here
    optimization: {
        minimize: false,
    },
}
```

Alternatively, if you are using the `UglifyJsPlugin`, you can tell it to not change class or function names like so:

```javascript
const UglifyJsPlugin = require("uglifyjs-webpack-plugin")

module.exports = {
    // ... other Webpack configurations here
    optimization: {
        minimizer: [
            new UglifyJsPlugin({
                uglifyOptions: {
                    keep_classnames: true,
                    keep_fnames: true,
                },
            }),
        ],
    },
}
```

Lastly, make sure in your data source options, the transpiled migration files are included:

```javascript
// TypeORM Configurations
module.exports = {
    // ...
    migrations: [
        // this is the relative path to the transpiled migration files in production
        "db/migrations/**/*.js",
        // your source migration files, used in development mode
        "src/db/migrations/**/*.ts",
    ],
}
```

## How to use Vite for the backend?

Using TypeORM in a Vite project is pretty straight forward. However, when you use migrations, you will run into "...migration name is wrong. Migration class name should have a
JavaScript timestamp appended." errors when running the production build.
On production builds, files are [optimized by default](https://vite.dev/config/build-options#build-minify) which includes mangling your code in order to minimize file sizes.

You have 3 options to mitigate this. The 3 options are shown belown as diff to this basic "vite.config.ts"

```typescript
import legacy from "@vitejs/plugin-legacy"
import vue from "@vitejs/plugin-vue"
import path from "path"
import { defineConfig } from "vite"

// https://vitejs.dev/config/
export default defineConfig({
    build: {
        sourcemap: true,
    },
    plugins: [vue(), legacy()],
    resolve: {
        alias: {
            "@": path.resolve(__dirname, "./src"),
        },
    },
})
```

### Option 1: Disable minify

This is the most crude option and will result in significantly larger files. Add `build.minify = false` to your config.

```diff
--- basic vite.config.ts
+++ disable minify vite.config.ts
@@ -7,6 +7,7 @@
 export default defineConfig({
   build: {
     sourcemap: true,
+    minify: false,
   },
   plugins: [vue(), legacy()],
   resolve: {
```

### Option 2: Disable esbuild minify identifiers

Vite uses esbuild as the default minifier. You can disable mangling of identifiers by adding `esbuild.minifyIdentifiers = false` to your config.
This will result in smaller file sizes, but depending on your code base you will get diminishing returns as all identifiers will be kept at full length.

```diff
--- basic vite.config.ts
+++ disable esbuild minify identifiers vite.config.ts
@@ -8,6 +8,7 @@
   build: {
     sourcemap: true,
   },
+  esbuild: { minifyIdentifiers: false },
   plugins: [vue(), legacy()],
   resolve: {
```

### Option 3: Use terser as minifier while keeping only the migration class names

Vite supports using terser as minifier. Terser is slower then esbuild, but offers more fine grained control over what to minify.
Add `minify: 'terser'` with `terserOptions.mangle.keep_classnames: /^Migrations\d+$/` and `terserOptions.compress.keep_classnames: /^Migrations\d+$/` to your config.
These options will make sure classnames that start with "Migrations" and end with numbers are not renamed during minification.

Make sure terser is available as dev dependency in your project: `npm add -D terser`.

```diff
--- basic vite.config.ts
+++ terser keep migration class names vite.config.ts
@@ -7,6 +7,11 @@
 export default defineConfig({
   build: {
     sourcemap: true,
+    minify: 'terser',
+    terserOptions: {
+      mangle: { keep_classnames: /^Migrations\d+$/ },
+      compress: { keep_classnames: /^Migrations\d+$/ },
+    },
   },
   plugins: [vue(), legacy()],
   resolve: {
```

## How to use TypeORM in ESM projects?

Make sure to add `"type": "module"` in the `package.json` of your project so TypeORM will know to use `import( ... )` on files.

To avoid circular dependency import issues use the `Relation` wrapper type for relation type definitions in entities:

```typescript
@Entity()
export class User {
    @OneToOne(() => Profile, (profile) => profile.user)
    profile: Relation<Profile>
}
```

Doing this prevents the type of the property from being saved in the transpiled code in the property metadata, preventing circular dependency issues.

Since the type of the column is already defined using the `@OneToOne` decorator, there's no use of the additional type metadata saved by TypeScript.

> Important: Do not use `Relation` on non-relation column types
