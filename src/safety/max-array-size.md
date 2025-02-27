Maximum Size of Arrays
======================

{{#include ../links.md}}

Rhai by default does not limit how large an [array] or a [BLOB] can be.

This can be changed via the [`Engine::set_max_array_size`][options] method, with zero being unlimited (the default).

A script attempting to create an [array] literal larger than the maximum will terminate with a parse error.

Any script operation that produces an [array] or a [BLOB] larger than the maximum also terminates
the script with an error result.

This check can be disabled via the [`unchecked`] feature for higher performance (but higher risks as well).

```rust
let mut engine = Engine::new();

engine.set_max_array_size(500);     // allow arrays only up to 500 items

engine.set_max_array_size(0);       // allow unlimited arrays
```

~~~admonish danger "Maximum size"

Be conservative when setting a maximum limit and always consider the fact that a registered function
may grow an [array]'s or [BLOB]'s size without Rhai noticing until the very end.

For instance, the built-in `+` operator for [arrays] and [BLOB's] concatenates two of them together
to form one larger [array] or [BLOB]; if both sources are _slightly_ below the maximum size limit,
the result may be almost _twice_ the maximum size.

As a malicious script may also create a deeply-nested [array] which consumes huge amounts of memory
while each individual [array] still stays under the maximum size limit, Rhai also _recursively_ adds
up the sizes of all [strings], [arrays], [blobs] and [object maps] contained within each [array] to
make sure that the _aggregate_ sizes of none of these data structures exceed their respective
maximum size limits (if any).

```rust
// Small, innocent array...
let small_array = [42];             // 1-deep... 1 item, 1 array

// ... becomes huge when multiplied!
small_array.push(small_array);      // 2-deep... 2 items, 2 arrays
small_array.push(small_array);      // 3-deep... 4 items, 4 arrays
small_array.push(small_array);      // 4-deep... 8 items, 8 arrays
small_array.push(small_array);      // 5-deep... 16 items, 16 arrays
          :
          :
small_array.push(small_array);      // <- Rhai raises an error somewhere here
small_array.push(small_array);      //    when the TOTAL number of items in
small_array.push(small_array);      //    the entire array tree exceeds limit

// Or this abomination...
let a = [ 42 ];

loop {
    a[0] = a;       // <- only 1 item, but infinite number of arrays
}
```
~~~
