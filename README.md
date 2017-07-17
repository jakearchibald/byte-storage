# Byte storage

The aim is to provide a low-level disk-backed storage system.

Developers should be able to use this to create their own higher-level storage systems, such as sql.

## API

```ts
self.byteStorage:ByteStorage;
```

The entry point for the API.

### Reading from a byte store

```ts
const readable:ReadableStream = await byteStorage.read(name:String, {
  start:Number = 0,
  end:Number
});
```

* `name` - the identifier of the store. This can be any string. `/` has no special meaning.
* `start` - start point within the store in bytes. If negative, treated as an offset from the end of the store.
* `end` - end point within the store in bytes. If negative, treated as an offset from the end of the store. If not provided, treated as the end of the store.

Resolves once:

* All write locks within intersecting byte ranges are released.
* A read lock is granted for the start-end.

Rejects if:

* `name` is not an existing byte store.
* The computed start point is less than 0.
* The computed start point is greater than the length of the byte store.
* The computed end point is less than 0.
* The computed end point is greater than the length of the byte store.
* The computed end point is less than the start point.

`readable` is a `ReadableString` with a underlying byte source. Non-BYOB reads produce `UInt8Array`s.

The read lock is released once the `readable` is closed.

### Writing to a byte store

```ts
const writable:WritableStream = await byteStorage.write(name:String, {
  start:Number = 0,
  end:Number
});
```

* `name` - the identifier of the store.
* `start` - start point within the store in bytes. If negative, treated as an offset from the end of the store.
* `end` - end point within the store in bytes. If negative, treated as an offset from the end of the store. If not provided, the store may continue to write beyond its current length, increasing its size.

Resolves once:

* The byte store entry is created, if not already.
* The space is allocated (zeroed), if `end` is provided and its computed value is greater than the length of the current store.
* The space is allocated (zeroed), if the computed start is greater than the length of the current store.
* All read and write locks for intersecting byte ranges are released.
* A write lock for the start-end (or end of the store) is granted.

Rejects if:

* The computed start point is less than 0.
* The computed end point is less than 0.
* The space cannot be allocated.

`writable` accepts chunks of `ArrayBuffer` or `ArrayBufferView`. Writing will error if additional allocation fails (this can only happen if `end` was not provided).

The `writable` will close once if `end` is provided, and `end - start` bytes have been queued.

If more than `end - start` bytes are queued, the writable errors.

If the readable closes before `end - start` bytes are queued, the writable closes. TODO: should we leave untouched bytes alone in this case?

If the readable errors, then any already-written bytes are retained, as in this is not a transactional system.

The write lock is released once the `writable` is closed.

### Transforming a byte store

```ts
byteStorage.transform(name:String, {
  readable:ReadableStream,
  writable:WritableStream
}, {
  start:Number = 0,
  end:Number
});
```

This functions the same as `.write` except:

* Writes to the writable are buffered if they're beyond the current read point (unless it's the end of the store). This means if you read 1 byte, and write 2, the next read in the transform will not include your written byte.
* Rejects if store `name` doesn't exist.

### Retrieving metadata on a byte store

```ts
const data = await byteStorage.status(name:String);

const {
  size:Number,
  created:Date,
  modified:Date
} = data;
```

* `name` - the identifier of the store.

Resolves once:

* All write locks for the store are released. TODO: or shall we just return the information we have, which may include half-written data?

`data` is `null` if the store does not exist.

### Resizing a byte store

```ts
await byteStorage.resize(name:String, end:Number);
```

* `name` - the identifier of the store.
* `end` - end point within the store in bytes. If negative, treated as an offset from the end of the store.

Resolves once:

* The space is allocated (zeroed), if `end` is greater than the length of the current store.
* The space is allocated (zeroed), if the computed start is greater than the length of the current store.
* All read and write locks for `end` until the end of the resource are released.
* A write lock for `end` until the end of the resource is granted.
* The space is allocated/deallocated.

Rejects if:

* `name` is not an existing byte store.
* The computed end point is less than 0.
* The space cannot be allocated.

### Deleting a byte store

```ts
const existed:Boolean = await byteStorage.delete(name:String);
```

Resolves once:

* All read and write locks for intersecting byte ranges are released.
* A write lock for the start-end (or end of the store) is granted.
* The store is unlinked.

TODO: or should we be more agressive here, and error current reads & writes?

### Getting all store names

We could add a `.keys()` method, or just use async iterators.

### Helpers for simple reads and writes?

```js
const data:UInt8Array = await byteStorage.readAll(name:String, {
  start:Number = 0,
  end:Number
});

await byteStorage.writeAll(name:String, data, {
  start:Number = 0
});
```

TODO: Do we need methods like above for making simple reads & writes?

## Issues

### Permalock

```js
await byteStorage.write('foo');
```

The above locks the whole of `"foo"` until the client closes. We could work around this by:

* Adding timeouts.
* Provide a method to discard existing locks (erroring the related open streams).

### Multi-action locking

Do we need an API to create locks independent of particular actions. Eg:

* I have a 500 byte store containing PNG data, I want to lock the whole store while I compress the data, which include reading, writing, and hopefully truncating.
* I am transforming some data, but I'm caching what I read. My write errors, but I want to write back what I originally read within the same lock.
