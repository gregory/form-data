# FormData

FormData implementation for Node.js

[![Code Coverage](https://codecov.io/github/octet-stream/form-data/coverage.svg?branch=master)](https://codecov.io/github/octet-stream/form-data?branch=master)
[![CI](https://github.com/octet-stream/form-data/workflows/CI/badge.svg)](https://github.com/octet-stream/form-data/actions/workflows/ci.yml)
[![ESLint](https://github.com/octet-stream/form-data/workflows/ESLint/badge.svg)](https://github.com/octet-stream/form-data/actions/workflows/eslint.yml)

## Installation

You can install this package with npm:

```
npm install formdata-node
```

Or yarn:

```
yarn add formdata-node
```

Or pnpm

```
pnpm add formdata-node
```

## Usage

1. Let's take a look at minimal example with [got](https://github.com/sindresorhus/got):

```js
import {FormData} from "formdata-node"

// I assume Got >= 12.x is used for this example
import got from "got"

const form = new FormData()

form.set("greeting", "Hello, World!")

got.post("https://httpbin.org/post", {body: form})
  .then(res => console.log("Res: ", res.body))
  .catch(err => console.error("Error: ", err))
```

2. If your HTTP client does not support spec-compliant FomrData, you can use [`form-data-encoder`](https://github.com/octet-stream/form-data-encoder) to encode entries:

```js
import {Readable} from "stream"

import {FormDataEncoder} from "form-data-encoder"
import {FormData} from "formdata-node"

// Note that `node-fetch` >= 3.x have builtin support for spec-compliant FormData, sou you'll only need the `form-data-encoder` if you use `node-fetch` <= 2.x.
import fetch from "node-fetch"

const form = new FormData()

form.set("field", "Some value")

const encoder = new FormDataEncoder(form)

const options = {
  method: "post",
  headers: encoder.headers,
  body: Readable.from(encoder)
}

await fetch("https://httpbin.org/post", options)
```

3. Sending files over form-data:

```js
import {FormData, File} from "formdata-node"

import fetch from "node-fetch"

const form = new FormData()
const file = new File(["My hovercraft is full of eels"], "hovercraft.txt")

form.set("file", file)

await fetch("https://httpbin.org/post", {method: "post", body: form})
```

4. Blobs as field's values allowed too:

```js
import {FormData} from "formdata-node"

import Blob from "fetch-blob"

const form = new FormData()
const blob = new Blob(["Some content"], {type: "text/plain"})

form.set("blob", blob)

// Will always be returned as `File`
let file = form.get("blob")

// The created file has "blob" as the name by default
console.log(file.name) // -> blob

// To change that, you need to set filename argument manually
form.set("file", blob, "some-file.txt")

file = form.get("file")

console.log(file.name) // -> some-file.txt
```

5. You can also append files using `fileFromPath` or `fileFromPathSync` helpers. It does the same thing as [`fetch-blob/from`](https://github.com/node-fetch/fetch-blob#blob-part-backed-up-by-filesystem), but returns a `File` instead of `Blob`:

```js
import {FormData, fileFromPath} from "formdata-node"

import fetch from "node-fetch"

const form = new FormData()

form.set("file", await fileFromPath("/path/to/a/file"))

await fetch("https://httpbin.org/post", {method: "post", body: form})
```

6. You can still use files sourced from any stream, but unlike in v2 you'll need some extra work to achieve that:

```js
import {Readable} from "stream"

import {FormData} from "formdata-node"

class BlobFromStream {
  #stream

  constructor(stream, size) {
    this.#stream = stream
    this.size = size
  }

  stream() {
    return this.#stream
  }

  get [Symbol.toStringTag]() {
    return "Blob"
  }
}

const content = Buffer.from("Stream content")

const stream = new Readable({
  read() {
    this.push(content)
    this.push(null)
  }
})

const form = new FormData()

form.set("stream", new BlobFromStream(stream, content.length), "file.txt")

await fetch("https://httpbin.org/post", {method: "post", body: form})
```

7. Note that if you don't know the length of that stream, you'll also need to handle form-data encoding manually or use [`form-data-encoder`](https://github.com/octet-stream/form-data-encoder) package. This is necessary to control which headers will be sent with your HTTP request:

```js
import {Readable} from "stream"

import {Encoder} from "form-data-encoder"
import {FormData} from "formdata-node"

const form = new FormData()

// You can use file-shaped or blob-shaped objects as FormData value instead of creating separate class
form.set("stream", {
  type: "text/plain",
  name: "file.txt",
  [Symbol.toStringTag]: "File",
  stream() {
    return getStreamFromSomewhere()
  }
})

const encoder = new Encoder(form)

const options = {
  method: "post",
  headers: {
    "content-type": encoder.contentType
  },
  body: Readable.from(encoder)
}

await fetch("https://httpbin.org/post", {method: "post", body: form})
```

## API

### `class FormData`

##### `constructor([entries]) -> {FormData}`

Creates a new FormData instance

  - **{array}** [entries = null] – an optional FormData initial entries.
    Each initial field should be passed as a collection of the objects
    with "name", "value" and "filename" props.
    See the [FormData#append()](#appendname-value-filename---void) for more info about the available format.

#### Instance methods

##### `set(name, value[, filename]) -> {void}`

Set a new value for an existing key inside **FormData**,
or add the new field if it does not already exist.

  - **{string}** name – The name of the field whose data is contained in **value**
  - **{unknown}** value – The field value. You can pass any JavaScript primitive type (including `null` and `undefined`), [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
    or [`File`](https://developer.mozilla.org/en-US/docs/Web/API/File).
    Note that Arrays and Object will be converted to **string** by using **String** function.
  - **{string}** [filename = undefined] – A filename of given field. Can be added only for `File` and `Blob`.

##### `append(name, value[, filename]) -> {void}`

Appends a new value onto an existing key inside a FormData object,
or adds the key if it does not already exist.

  - **{string}** name – The name of the field whose data is contained in **value**
  - **{unknown}** value – The field value. You can pass any JavaScript primitive type (including `null` and `undefined`), [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
    or [`File`](https://developer.mozilla.org/en-US/docs/Web/API/File).
    Note that Arrays and Object will be converted to **string** by using **String** function.
  - **{string}** [filename = undefined] – A filename of given field. Can be added only for `File` and `Blob`.

##### `get(name) -> {string | File}`

Returns the first value associated with the given name.
If the field has `Blob`, `Buffer`, `File` or `ReadStream` value, the File-like object will be returned.

  - **{string}** name – A name of the value you want to retrieve.

##### `getAll(name) -> {Array<string | File>}`

Returns all the values associated with a given key from within a **FormData** object.
If the field has `Blob`, `Buffer`, `File` or `ReadStream` value, the File-like object will be returned.

  - **{string}** name – A name of the value you want to retrieve.

##### `has(name) -> {boolean}`

Check if a field with the given **name** exists inside **FormData**.

  - **{string}** – A name of the field you want to test for.

##### `delete(name) -> {void}`

Deletes a key and its value(s) from a `FormData` object.

  - **{string}** name – The name of the key you want to delete.

##### `forEach(callback[, ctx]) -> {void}`

Executes a given **callback** for each field of the FormData instance

  - **{function}** callback – Function to execute for each element, taking three arguments:
    + **{string | File}** value – A value(s) of the current field.
    + **{string}** name – Name of the current field.
    + **{FormData}** fd – The FormData instance that **forEach** is being applied to
  - **{unknown}** [ctx = null] – Value to use as **this** context when executing the given **callback**

##### `keys() -> {Generator<string>}`

Returns an [`iterator`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) allowing to go through the **FormData** keys

##### `values() -> {Generator<string | File>}`

Returns an [`iterator`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) allowing to go through the **FormData** values

##### `entries() -> {Generator<[string, string | File]>}`

Returns an [`iterator`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) allowing to go through the **FormData** key/value pairs

##### `[Symbol.iterator]() -> {Generator<[string, string | File]>}`

An alias for [`FormData#entries()`](#entries---iterator)

### `class File extends Blob`

##### `constructor(blobParts, filename[, options]) -> {File}`

The `File` class provides information about files. The `File` object inherits `Blob` from [`fetch-blob`](https://github.com/bitinn/fetch-blob) package.

  - **{(ArrayBufferLike | ArrayBufferView | Blob | string)[]}** blobParts
  - **{string}** filename – Representing the file name.
  - **{object}** [options = {}] - An options object containing optional attributes for the file. Available options are as follows
  - **{number}** [options.lastModified = Date.now()] – provides the last modified date of the file as the number of milliseconds since the Unix epoch (January 1, 1970 at midnight). Files without a known last modified date return the current date.
  - **{string}** [options.type = ""] - Returns the media type ([`MIME`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)) of the file represented by a `File` object.

### `fileFromPath(path[, filename, options]) -> {Promise<File>}`

Creates a `File` referencing the one on a disk by given path.

  - **{string}** path - Path to a file
  - **{string}** [filename] - Name of the file. Will be passed as second argument in `File` constructor. If not presented, the file path will be used to get it.
  - **{object}** [options = {}] - File options.
  - **{string}** [options.type = ""] - Returns the media type ([`MIME`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)) of the file represented by a `File` object.

### `fileFromPathSync(path[, filename, options]) -> {File}`

Creates a `File` referencing the one on a disk by given path. Synchronous version of the `fileFromPath`.

  - **{string}** path - Path to a file
  - **{string}** [filename] - Name of the file. Will be passed as second argument in `File` constructor. If not presented, the file path will be used to get it.
  - **{object}** [options = {}] - File options.
  - **{string}** [options.type = ""] - Returns the media type ([`MIME`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)) of the file represented by a `File` object.

### `isFileLike(value) -> {boolean}`

Check if given value is a File, Blob or file-look-a-like object.

  - **{unknown}** value - A value to test

## Related links

- [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData) documentation on MDN
- [`File`](https://developer.mozilla.org/en-US/docs/Web/API/File) documentation on MDN
- [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob) documentation on MDN
- [`formdata-polyfill`](https://github.com/jimmywarting/FormData) HTML5 `FormData` for Browsers & NodeJS.
- [`fetch-blob`](https://github.com/node-fetch/fetch-blob) a Blob implementation on node.js, originally from node-fetch.
- [`form-data-encoder`](https://github.com/octet-stream/form-data-encoder) - Encoder for multipart/form-data
- [`then-busboy`](https://github.com/octet-stream/then-busboy) a promise-based wrapper around Busboy. Process multipart/form-data content and returns it as a single object. Will be helpful to handle your data on the server-side applications.
- [`@octetstream/object-to-form-data`](https://github.com/octet-stream/object-to-form-data) converts JavaScript object to FormData.
