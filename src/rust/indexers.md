Custom Type Indexers
====================

{{#include ../links.md}}

A [custom type] can also expose an _indexer_ by registering an indexer function.

A [custom type] with an indexer function defined can use the bracket notation to get/set a property
value at a particular index:

> _object_ `[` _index_ `]`
>
> _object_ `[` _index_ `]` `=` _value_ `;`

Like property [getters/setters], indexers take a `&mut` reference to the first parameter.

They also take an additional parameter of any type that serves as the _index_ within brackets.

Indexers are disabled when the [`no_index`] and [`no_object`] features are used together.

| `Engine` API                  | Function signature(s)<br/>(`T: Clone` = custom type,<br/>`X: Clone` = index type,<br/>`V: Clone` = data type) |        Can mutate `T`?         |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------- | :----------------------------: |
| `register_indexer_get`        | `Fn(&mut T, X) -> V`                                                                                          |      yes, but not advised      |
| `register_indexer_set`        | `Fn(&mut T, X, V)`                                                                                            |              yes               |
| `register_indexer_get_set`    | getter: `Fn(&mut T, X) -> V`<br/>setter: `Fn(&mut T, X, V)`                                                   | yes, but not advised in getter |
| `register_indexer_get_result` | `Fn(&mut T, X) -> Result<Dynamic, Box<EvalAltResult>>`                                                        |      yes, but not advised      |
| `register_indexer_set_result` | `Fn(&mut T, X, V) -> Result<(), Box<EvalAltResult>>`                                                          |              yes               |

```admonish danger.small "No support for references"

Rhai does NOT support normal references (i.e. `&T`) as parameters.
All references must be mutable (i.e. `&mut T`).
```

```admonish warning.small "Getters must be pure"

By convention, index getters are not supposed to mutate the [custom type],
although there is nothing that prevents this mutation.
```


Cannot Override Arrays, BLOB's, Object Maps, Strings and Integers
-----------------------------------------------------------------

```admonish failure.side "Plugins"

They can be defined in a [plugin module], but will be ignored.
```

For efficiency reasons, indexers **cannot** be used to overload (i.e. override)
built-in indexing operations for [arrays], [object maps], [strings] and integers
(acting as [bit-field] operation).

The following types have built-in indexer implementations that are fast and efficient.

<section></section>

| Type                                      |                Index type                 | Return type | Description                                                                  |
| ----------------------------------------- | :---------------------------------------: | :---------: | ---------------------------------------------------------------------------- |
| [`Array`]                                 |                   `INT`                   | [`Dynamic`] | access a particular element inside the [array]                               |
| [`Blob`]                                  |                   `INT`                   |    `INT`    | access a particular byte value inside the [BLOB]                             |
| [`Map`]                                   | [`ImmutableString`],<br/>`String`, `&str` | [`Dynamic`] | access a particular property inside the [object map]                         |
| [`ImmutableString`],<br/>`String`, `&str` |                   `INT`                   | [character] | access a particular [character] inside the [string]                          |
| `INT`                                     |                   `INT`                   |   boolean   | access a particular bit inside the integer number as a [bit-field]           |
| `INT`                                     |                  [range]                  |    `INT`    | access a particular range of bits inside the integer number as a [bit-field] |

```admonish warning.small "Do not overload indexers for built-in standard types"

In general, it is a bad idea to overload indexers for any of the [standard types] supported
internally by Rhai, since built-in indexers may be added in future versions.
```


Examples
--------

```rust
#[derive(Debug, Clone)]
struct TestStruct {
    fields: Vec<i64>
}

impl TestStruct {
    // Remember &mut must be used even for getters
    fn get_field(&mut self, index: String) -> i64 {
        self.fields[index.len()]
    }
    fn set_field(&mut self, index: String, value: i64) {
        self.fields[index.len()] = value
    }

    fn new() -> Self {
        Self { fields: vec![1, 2, 3, 4, 5] }
    }
}

let mut engine = Engine::new();

engine.register_type::<TestStruct>()
      .register_fn("new_ts", TestStruct::new)
      // Short-hand: .register_indexer_get_set(TestStruct::get_field, TestStruct::set_field);
      .register_indexer_get(TestStruct::get_field)
      .register_indexer_set(TestStruct::set_field);

let result = engine.eval::<i64>(
r#"
    let a = new_ts();
    a["xyz"] = 42;                  // these indexers use strings
    a["xyz"]                        // as the index type
"#)?;

println!("Answer: {}", result);     // prints 42
```


Convention for Negative Index
-----------------------------

If the indexer takes a signed integer as an index (e.g. the standard `INT` type), care should be
taken to handle _negative_ values passed as the index.

It is a standard API _convention_ for Rhai to assume that an index position counts _backwards_ from
the _end_ if it is negative.

`-1` as an index usually refers to the _last_ item, `-2` the second to last item, and so on.

Therefore, negative index values go from `-1` (last item) to `-length` (first item).

A typical implementation for negative index values is:

```rust
// The following assumes:
//   'index' is 'INT', 'items: usize' is the number of elements
let actual_index = if index < 0 {
    index.checked_abs().map_or(0, |n| items - (n as usize).min(items))
} else {
    index as usize
};
```

The _end_ of a data type can be interpreted creatively.  For example, in an integer used as a
[bit-field], the _start_ is the _least-significant-bit_ (LSB) while the `end` is the
_most-significant-bit_ (MSB).


Convention for Range Index
--------------------------

```admonish tip.side.wide "Tip: Negative values"

By convention, negative values are _not_ interpreted specially in indexers for [ranges].
```

It is very common for [ranges] to be used as indexer parameters via the types
`std::ops::Range<INT>` (exclusive) and `std::ops::RangeInclusive<INT>` (inclusive).

One complication is that two versions of the same indexer must be defined to support _exclusive_
and _inclusive_ [ranges] respectively.

```rust
use std::ops::{Range, RangeInclusive};

let mut engine = Engine::new();

engine
    /// Version of indexer that accepts an exclusive range
    .register_indexer_get_set(
        |obj: &mut TestStruct, range: Range<i64>| -> bool { ... },
        |obj: &mut TestStruct, range: Range<i64>, value: bool| { ... },
    )
    /// Version of indexer that accepts an inclusive range
    .register_indexer_get_set(
        |obj: &mut TestStruct, range: RangeInclusive<i64>| -> bool { ... },
        |obj: &mut TestStruct, range: RangeInclusive<i64>, value: bool| { ... },
    );

engine.run(
"
    let obj = new_ts();

    let x = obj[0..12];             // use exclusive range

    obj[0..=11] = !x;               // use inclusive range
")?;
```


Indexer as Property Access Fallback
-----------------------------------

```admonish tip.side.wide "Tip: Property bag"

Such an indexer allows easy creation of _property bags_ (similar to [object maps])
which can dynamically add/remove properties.
```

An indexer taking a [string] index is a special case &ndash; it acts as a _fallback_ to property
[getters/setters].

During a property access, if the appropriate property [getter/setter][getters/setters] is not
defined, an indexer is called and passed the string name of the property.

This is also extremely useful as a short-hand for indexers, when the [string] keys conform to
property name syntax.

```rust
// Assume 'obj' has an indexer defined with string parameters...

// Let's create a new key...
obj.hello_world = 42;

// The above is equivalent to this:
obj["hello_world"] = 42;

// You can write this...
let x = obj["hello_world"];

// but it is easier with this...
let x = obj.hello_world;
```

### Caveat &ndash; reverse is NOT true

The reverse, however, is not true &ndash; when an indexer fails or doesn't exist, the corresponding
property [getter/setter][getters/setters], if any, is not called.

```rust
type MyType = HashMap<String, i64>;

let mut engine = Engine::new();

// Define custom type, property getter and string indexers
engine.register_type::<MyType>()
      .register_fn("new_ts", || {
          let mut obj = MyType::new();
          obj.insert("foo".to_string(), 1);
          obj.insert("bar".to_string(), 42);
          obj.insert("baz".to_string(), 123);
          obj
      })
      // Property 'hello'
      .register_get("hello", |obj: &mut MyType| obj.len() as i64)
      // Index getter/setter
      .register_indexer_get_result(|obj: &mut MyType, prop: &str|
          obj.get(index).cloned().ok_or_else(|| "not found".into())
      ).register_indexer_set(|obj: &mut MyType, prop: &str, value: i64|
          obj.insert(prop.to_string(), value)
      );

engine.run("let ts = new_ts(); print(ts.foo);");
//                                   ^^^^^^
//                 Calls ts["foo"] - getter for 'foo' does not exist

engine.run("let ts = new_ts(); print(ts.bar);");
//                                   ^^^^^^
//                 Calls ts["bar"] - getter for 'bar' does not exist

engine.run("let ts = new_ts(); ts.baz = 999;");
//                             ^^^^^^^^^^^^
//                 Calls ts["baz"] = 999 - setter for 'baz' does not exist

engine.run(r#"let ts = new_ts(); print(ts["hello"]);"#);
//                                     ^^^^^^^^^^^
//                 Error: Property getter for 'hello' not a fallback for indexer
```
