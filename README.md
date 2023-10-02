# @metalsmith/permalinks

A Metalsmith plugin that applies a custom permalink pattern to files, and renames them so that they're nested properly for static sites (converting `about.html` into `about/index.html`).

[![metalsmith: core plugin][metalsmith-badge]][metalsmith-url]
[![npm: version][npm-badge]][npm-url]
[![ci: build][ci-badge]][ci-url]
[![code coverage][codecov-badge]][codecov-url]
[![license: MIT][license-badge]][license-url]

## Installation

NPM:

```bash
npm install @metalsmith/permalinks
```

Yarn:

```bash
yarn add @metalsmith/permalinks
```

## Usage

```js
import { dirname } from 'path'
import { fileURLToPath } from 'url'
import Metalsmith from 'metalsmith'
import permalinks from '@metalsmith/permalinks'

const __dirname = dirname(fileURLToPath(import.meta.url))

Metalsmith(__dirname).use(
  permalinks({
    pattern: ':title'
  })
)
```

The `pattern` can contain a reference to **any piece of metadata associated with the file** by using the `:PROPERTY` syntax for placeholders.
By default, all files get a `:dirname/:basename` (+ directoryIndex = `/index.html`) pattern, i.e. the original filepath `blog/post1.html` becomes `blog/post1/index.html`. The `dirname` and `basename` values are automatically made available by @metalsmith/permalinks for the purpose of generating the permalink.

If no pattern is provided, the files won't be remapped, but the `permalink` metadata key will still be set, so that you can use it for outputting links to files in the template.

The `pattern` can also be set as such:

```js
metalsmith.use(
  permalinks({
    // original options would act as the keys of a `default` linkset,
    pattern: ':title',
    date: 'YYYY',

    // each linkset defines a match, and any other desired option
    linksets: [
      {
        match: { collection: 'blogposts' },
        pattern: 'blog/:date/:title',
        date: 'mmddyy'
      },
      {
        match: { collection: 'pages' },
        pattern: 'pages/:title'
      }
    ]
  })
)
```

### Optional permalink pattern parts

The pattern option can also contain optional placeholders with the syntax `:PROPERTY?`. If the property is not defined in a file's metadata, it will be replaced with an empty string `''`. For example the pattern `:category?/:title` applied to a source directory with 2 files:

<table>
  <tr>
    <td>
<pre><code>---
title: With category
category: category1
---</pre></code>
    </td>
    <td>
<pre><code>---
title: No category
---</pre></code>
    </td>
  </tr>
</table>

would generate the file tree:

```
build
├── category1/with-category/index.html
└── no-category/index.html
```

### Dates

By default any date will be converted to a `YYYY/MM/DD` format when using in a permalink pattern, but you can change the conversion by passing a `date` option:

```js
metalsmith.use(
  permalinks({
    pattern: ':date/:title',
    date: 'YYYY'
  })
)
```

It uses [moment.js](https://momentjs.com/docs/#/displaying/format/) to format the string.

### Slug options

You can finetune how a pattern is processed by providing custom [slug](https://developer.mozilla.org/en-US/docs/Glossary/Slug) options.
By default [slugify](https://www.npmjs.com/package/slugify) is used and patterns will be lowercased.

You can pass custom [slug options](https://www.npmjs.com/package/slugify#options):

```js
metalsmith.use(
  permalinks({
    slug: {
      replacement: '_',
      lower: false
    }
  })
)
```

The following makes everything snake-case but allows `'` to be converted to `-`

```js
metalsmith.use(
  permalinks({
    slug: {
      remove: /[^a-z0-9- ]+/gi,
      lower: true,
      extend: {
        "'": '-'
      }
    }
  })
)
```

#### Handling special characters

If your pattern parts contain special characters like `:` or `=`, specifying `slug.strict` as `true` is a quick way to remove them:

```js
metalsmith.use(
  permalinks({
    slug: {
      lower: true,
      strict: true
    }
  })
)
```

#### Custom 'slug' function

If the result is not to your liking, you can replace the slug function altogether.
For now only the js version of syntax is supported and tested.

```js
metalsmith.use(
  permalinks({
    pattern: ':title',
    slug: require('transliteration').slugify
  })
)
```

There are plenty of other options on npm for transliteration and slugs. <https://www.npmjs.com/browse/keyword/transliteration>.

### Skipping Permalinks for a file

A file can be ignored by the permalinks plugin if you pass the `permalink: false` option to the yaml metadata of a file.
This is useful for hosting a static site on AWS S3, where there is a top level `error.html` file and not an `error/index.html` file.

For example, in your error.md file:

```js
---
template: error.html
title: error
permalink: false
---
```

### Overriding the permalink for a file

Using the `permalink` property in a file's front-matter, its permalink can be overridden. This can be useful for transferring
projects over to Metalsmith where pages don't follow a strict permalink system.

For example, in one of your pages:

```js
---
title: My Post
permalink: "posts/my-post"
---
```

### Overriding the default `index.html` file

Use `indexFile` to define a custom index file.

```js
metalsmith.use(
  permalinks({
    indexFile: 'alt.html'
  })
)
```

### Ensure files have unique URIs

Normally you should take care to make sure your source files do not permalink to the same target.  
When URI clashes occur nevertheless, the build will halt with an error stating the target file conflict.

```js
metalsmith.use(
  permalinks({
    duplicates: 'error'
  })
)
```

There are 3 other possible values for the `duplicates` option: `index` will add an `-<index>` suffix to other files with the same target URI, , `overwrite` will silently overwrite previous files with the same target URI.

The third possibility is to provide your own function to handle duplicates, with the signature:

```js
function paginateDupes(targetPath, files, filename, options) => {
  let target,
    counter = 0,
    postfix = ''
  while (files[target]) {
    postfix = `/${++counter}`
    target = path.join(`${targetPath}${postfix}`, options.indexFile)
  }
  return target
}
```

Return an error in the custom duplicates handler to halt the build.  
The example above is a variant of the `index` value, where 2 files targeting the URI `gallery` will be written to `gallery/1/index.html` and `gallery/2/index.html`.

_Note_: The `duplicates` option combines the `unique` and `duplicatesFail` options of version < 2.4.1. Specifically, `duplicatesFail:true` maps to `duplicates:'error'`, `unique:true` maps to `duplicates:'index'`, and `unique:false` or `duplicatesFail:false` map to `duplicates:'overwrite'`.

### Debug

To enable debug logs, set the `DEBUG` environment variable to `@metalsmith/permalinks`:

```js
metalsmith.env('DEBUG', '@metalsmith/permalinks*')
```

Alternatively you can set `DEBUG` to `@metalsmith/*` to debug all Metalsmith core plugins.

### CLI usage

To use this plugin with the Metalsmith CLI, add `@metalsmith/permalinks` to the `plugins` key in your `metalsmith.json` file:

```json
{
  "plugins": [
    {
      "permalinks": {
        "pattern": ":title"
      }
    }
  ]
}
```

## License

[MIT](LICENSE)

[npm-badge]: https://img.shields.io/npm/v/@metalsmith/permalinks.svg
[npm-url]: https://www.npmjs.com/package/@metalsmith/permalinks
[ci-badge]: https://github.com/metalsmith/permalinks/actions/workflows/test.yml/badge.svg
[ci-url]: https://github.com/metalsmith/permalinks/actions/workflows/test.yml
[metalsmith-badge]: https://img.shields.io/badge/metalsmith-plugin-green.svg?longCache=true
[metalsmith-url]: https://metalsmith.io/
[codecov-badge]: https://img.shields.io/coveralls/github/metalsmith/permalinks
[codecov-url]: https://coveralls.io/github/metalsmith/permalinks
[license-badge]: https://img.shields.io/github/license/metalsmith/permalinks
[license-url]: LICENSE
