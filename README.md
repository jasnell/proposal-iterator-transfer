# Iterator Transfer

A proposal to add a `transfer()` method to sync and async iterators. The method
returns a new iterator that takes over the original iterator's state, and
leaves the original iterator closed.

`transfer()` cuts off every existing reference to the original iterator. It
does not stop the returned iterator from being shared. Code that uses
`transfer()` still has to make sure the returned iterator reaches its intended
consumer and nobody else. See
[What `transfer()` does not guarantee](#what-transfer-does-not-guarantee).

## Status

**Stage:** 0

**Champion(s):** James M Snell (@jasnell)

**Author(s):** James M Snell (@jasnell)

This proposal has not yet been presented to TC39.

## Motivation

Many APIs that consume an iterator expect to be its only consumer. Some
examples:

- `ReadableStream.from(iterable)` turns an iterator into a stream. The stream
  can be locked to a single reader.
- `Iterator.from(...)` and iterator helpers (`.map()`, `.filter()`,
  `.take()`, ...) wrap an underlying iterator.
- Library code that buffers, batches, or pipes values from an iterator
  somewhere else.

The language gives code that hands off an iterator no way to cut off other
references to it. Code that still holds a reference to the original iterator can keep calling
`next()`, `return()`, or `throw()`. Values then go to the wrong consumer, and
the "exclusive" consumer ends up with a damaged view of the sequence.

### The problem

```js
// Take a generator function...
function* gen() {
  yield 'a';
  yield 'b';
}

// ...and get an iterator from it.
const g = gen();

// At this point, the ReadableStream assumes it owns the iterator...
const rs = ReadableStream.from(g);
const reader = rs.getReader();

// ...but the generator can still be used, even though the stream is locked.
g.next();                         // Consumes 'a'
reader.read().then(console.log);  // Consumes 'b'
reader.read().then(console.log);  // done!
```

The stream is locked to `reader`, but that lock only protects the stream. It
does not protect the iterator underneath it. Anyone who holds `g` can take
values out of the stream's source without the stream knowing.

The same thing happens with any wrapper:

```js
const g = gen();
const upper = g.map((s) => s.toUpperCase());

g.next();     // { value: 'a', done: false }  -- taken from under `upper`
upper.next(); // { value: 'B', done: false }  -- 'A' is missing
```

Today, a consumer that needs exclusive access has two options, and neither is
good:

1. **Trust the caller** not to touch the iterator again. Nothing enforces
   this, and the bugs it causes are subtle and depend on timing (especially
   with async iterators).
2. **Eagerly drain** the iterator into a buffer. This defeats laziness, breaks
   on infinite sequences, and loses backpressure.

### The proposed solution

`transfer()` moves the iterator's internal state into a new iterator object.
After the transfer, the original object is closed: it reports `done` and has
nothing left to iterate. At the moment it is returned, the new object is the
only handle to the remaining sequence.

```js
// Take a generator function...
function* gen() {
  yield 'a';
  yield 'b';
}

// ...and get an iterator from it.
const g = gen();

// Pass the transferred iterator straight to the stream, keeping no
// reference to it.
const rs = ReadableStream.from(g.transfer());
const reader = rs.getReader();

// The original generator can no longer interfere.
g.next();                         // { value: undefined, done: true }
reader.read().then(console.log);  // Consumes 'a'
reader.read().then(console.log);  // Consumes 'b'
```

This example works because the returned iterator goes straight into
`ReadableStream.from()` and is never stored anywhere else. If the caller kept a
reference to it, the original problem would come back (see
[What `transfer()` does not guarantee](#what-transfer-does-not-guarantee)).

This works like `ArrayBuffer.prototype.transfer()`, which moves ownership of a
buffer's contents to a new `ArrayBuffer` and detaches the original. Here,
ownership of an iteration moves instead of ownership of memory. Both have the
same limit: whoever holds the new object can share it.

## API

`transfer()` is a method on the iterator prototypes. This matches
`ArrayBuffer.prototype.transfer()`.

```js
Iterator.prototype.transfer()          // returns a new iterator
%AsyncIteratorPrototype%.transfer()    // returns a new async iterator
```

### Semantics

- `transfer()` returns a new iterator object that continues the sequence from
  where the original left off.
- The new iterator has the same prototype as the original. A transferred
  generator is still a generator object, a transferred array iterator is still
  an array iterator, and so on.

  ```js
  const g = gen();
  const t = g.transfer();

  Object.getPrototypeOf(t) === Object.getPrototypeOf(g); // true
  ```

- After `transfer()`, the original iterator acts as if it has finished:
  - `next()` returns `{ value: undefined, done: true }`.
  - `return()` returns `{ value: undefined, done: true }` and does **not** run
    cleanup (such as `finally` blocks in a generator). Cleanup now belongs to
    the new iterator.
  - `throw(e)` behaves as it would on a completed iterator.
- The transfer itself does not advance the iterator or run any of its code.
- Transferring an iterator that is already closed returns a closed iterator.

### What `transfer()` does not guarantee

`transfer()` invalidates references to the *original* iterator. It does
nothing about references to the *returned* iterator. The returned iterator is
an ordinary object, and any code that holds it can call `next()`, `return()`,
or `throw()` on it. JavaScript has no ownership checks to prevent this.

So `transfer()` alone doesn't make shared use impossible. This code has the
same bug as the original example:

```js
const g = gen();
const t = g.transfer();

const rs = ReadableStream.from(t);
const reader = rs.getReader();

t.next();                         // Consumes 'a' -- `t` is shared
reader.read().then(console.log);  // Consumes 'b'
```

For the hand-off to be correct, users still need to prevent shared use of the
returned iterator:

- **The giver** should pass the result of `transfer()` straight to the
  consumer, as in `ReadableStream.from(g.transfer())`, and not keep or share
  the returned iterator.
- **A consumer** that wants to protect itself from callers who keep a
  reference can call `transfer()` on the iterator it receives, and use only
  the result. This cuts off everyone else who holds the iterator it was given.
  The consumer is then responsible for not leaking the iterator it now holds.

What `transfer()` adds is a reliable way to cut off earlier references at a
known point. Before that point, every holder can use the iterator. After it,
only the returned iterator can continue the sequence. Keeping the returned
iterator from being shared is still up to the code that holds it.

### Only inactive iterators can be transferred

`transfer()` throws a `TypeError` if the iterator is active. An iterator is
active if:

- it is a generator that is currently running. For example, `g.transfer()` is
  called from inside `g`'s own body. A re-entrant `next()` call throws in the
  same way today.
- it is an async iterator with pending requests, such as `next()` calls whose
  promises have not settled yet.

```js
async function* agen() {
  yield 'a';
}

const ag = agen();
const p = ag.next();  // request is pending

ag.transfer();        // TypeError: cannot transfer an active iterator

await p;
const t = ag.transfer(); // OK: no pending requests
```

This keeps the hand-off clean. Every request made on the original iterator
settles on the original iterator, and the new owner starts with an empty
queue.

### Iterator helpers

Transferring an iterator helper transfers only the helper. The helper's state,
including its reference to the iterator underneath it, moves to the new
helper. The underlying iterator is not transferred.

```js
const g = gen();
const upper = g.map((s) => s.toUpperCase());
const t = upper.transfer();

upper.next(); // { value: undefined, done: true }
t.next();     // { value: 'A', done: false }
g.next();     // { value: 'b', done: false }  -- `g` is still reachable
```

To cut off references to every iterator in the chain, transfer the source
before wrapping it. As above, the result is only exclusive if the code that
holds `upper` doesn't share it.

```js
const upper = g.transfer().map((s) => s.toUpperCase());
```

### Transferring user-defined iterators

Built-in iterators (generators, async generators, array, map, set, and string
iterators, and iterator helpers) keep their state in internal slots, so the
engine can move that state directly.

User-defined iterators that inherit from `Iterator.prototype` keep their state
in places the engine can't see, such as their own properties or closures. This
proposal documents three approaches for these iterators:

1. **Built-ins only.** `transfer()` throws a `TypeError` on iterators that
   don't have the internal slots of a built-in iterator. This is the simplest
   approach and has no way to fail silently, but it makes `transfer()`
   unusable for user-defined iterators.
2. **A transfer protocol.** User-defined iterators implement a hook, for
   example a well-known symbol method, that `transfer()` calls to move their
   state to a new object and close the original. This gives user-defined
   iterators the same guarantees as built-ins, as long as the author
   implements the hook correctly.
3. **A generic fallback.** For iterators that are neither built-in nor
   implement the protocol, `transfer()` returns a new iterator that forwards
   to the original. This works for any iterator, but the engine can't close
   the original, so the new iterator only has exclusive ownership if no other
   code calls the original iterator's methods directly.

## Non-goals

This proposal does not change any APIs that consume iterators, such as
`ReadableStream.from()`, `Iterator.from()`, or `Array.fromAsync()`. Whether
those APIs should transfer the iterators they receive is a separate question
for those APIs.

## Prior art and related work

- [`ArrayBuffer.prototype.transfer()`](https://github.com/tc39/proposal-arraybuffer-transfer)
  moves ownership of a buffer and detaches the original.
- `structuredClone(value, { transfer })` and `postMessage` transfer lists on
  the web platform.
- [WHATWG Streams](https://streams.spec.whatwg.org/) reader locking, which
  gives a stream an exclusive consumer but does not extend that guarantee to
  the stream's source.
- Move semantics and ownership transfer in languages such as Rust and C++.
  Rust's compiler rejects any use of a value after it has been moved. C++
  leaves that to convention. JavaScript has no compile-time checks, so
  `transfer()` closes the original at runtime and, like C++, relies on users
  not to share the new object.
- [Iterator helpers](https://github.com/tc39/proposal-iterator-helpers) and
  [async iterator helpers](https://github.com/tc39/proposal-async-iterator-helpers),
  which wrap iterators but cannot stop other code from using the underlying
  iterator.

## Specification

The specification text has not been written yet. Once it exists, it will be
in [`spec.emu`](./spec.emu).
