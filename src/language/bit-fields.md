Integer as Bit-Fields
=====================

{{#include ../links.md}}

```admonish note.side.wide

Nothing here cannot be done via standard bit-manipulation (i.e. shifting and masking).

Built-in support is more elegant and performant since it usually replaces a sequence of multiple steps.

```

Since bit-wise operators are defined on integer numbers, individual bits can also be accessed and
manipulated via an indexing syntax.

If a bit is set (i.e. `1`), the index access returns `true`.

If a bit is not set (i.e. `0`), the index access returns `false`.

When a [range] is used, the bits within the [range] are shifted and extracted as an integer value.

Bit-fields are very commonly used in embedded systems which must squeeze data into limited memory.
Built-in support makes handling them efficient.

Indexing an integer as a bit-field is disabled for the [`no_index`] feature.


Syntax
------

### From Least-Significant Bit (LSB)

Bits in a bit-field are accessed with zero-based, non-negative integer indices:

> _integer_ `[` _index from 0 to 63 or 31_ `]`
>
> _integer_ `[` _index from 0 to 63 or 31_ `] =` `true` or `false` ;

[Ranges] can also be used:

> _integer_ `[` _start_ `..` _end_ `]`  
> _integer_ `[` _start_ `..=` _end_ `]`
>
> _integer_ `[` _start_ `..` _end_ `] =` _new integer value_ ;  
> _integer_ `[` _start_ `..=` _end_ `] =` _new integer value_ ;

```admonish warning.small "Number of bits"

The maximum bit number that can be accessed is 63 (or 31 under [`only_i32`]).

Bits outside of the range are ignored.
```


### From Most-Significant Bit (MSB)

A _negative_ index accesses a bit in the bit-field counting from the _end_, or from the
_most-significant bit_, with &minus;1 being the _highest_ bit.

> _integer_ `[` _index from &minus;1 to &minus;64 or &minus;32_ `]`
>
> _integer_ `[` _index from &minus;1 to &minus;64 or &minus;32_ `] =` `true` or `false` ;

[Ranges] always count from the least-significant bit (LSB) and has no support for negative positions.

```admonish warning.small "Number of bits"

The maximum bit number that can be accessed is &minus;64 (or &minus;32 under [`only_i32`]).
```


Bit-Field Functions
-------------------

The following standard functions (defined in the [`BitFieldPackage`][built-in packages] but excluded if
using a [raw `Engine`]) operate on `INT` bit-fields.

These functions are available even under the [`no_index`] feature.

| Function                   | Parameter(s)                                                                                                                                                   | Description                                               |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `get_bit`                  | bit number, counting from MSB if < 0                                                                                                                           | returns the state of a bit: `true` if `1`, `false` if `0` |
| `set_bit`                  | <ol><li>bit number, counting from MSB if < 0</li><li>new state: `true` if `1`, `false` if `0`</li></ol>                                                        | sets the state of a bit                                   |
| `get_bits`                 | <ol><li>starting bit number, counting from MSB if < 0</li><li>number of bits to extract, none if < 1, to MSB if ≥ _length_</li></ol>                           | extracts a number of bits, shifted towards LSB            |
| `get_bits`                 | [range] of bits                                                                                                                                                | extracts a number of bits, shifted towards LSB            |
| `set_bits`                 | <ol><li>starting bit number, counting from MSB if < 0</li><li>number of bits to set, none if < 1, to MSB if ≥ _length_<br/>3) new value</li></ol>              | sets a number of bits from the new value                  |
| `set_bits`                 | <ol><li>[range] of bits</li><li>new value</li></ol>                                                                                                            | sets a number of bits from the new value                  |
| `bits` method and property | <ol><li>_(optional)_ starting bit number, counting from MSB if < 0</li><li>_(optional)_ number of bits to extract, none if < 1, to MSB if ≥ _length_</li></ol> | allows iteration over the bits of a bit-field             |
| `bits`                     | [range] of bits                                                                                                                                                | allows iteration over the bits of a bit-field             |


Example
-------

```js , no_run
// Assume the following bits fields in a single 16-bit word:
// ┌─────────┬────────────┬──────┬─────────┐
// │  15-12  │    11-4    │  3   │   2-0   │
// ├─────────┼────────────┼──────┼─────────┤
// │    0    │ 0-255 data │ flag │ command │
// └─────────┴────────────┴──────┴─────────┘

let value = read_start_hw_register(42);

let command = value.get_bits(0, 3);         // Command = bits 0-2

let flag = value[3];                        // Flag = bit 3

let data = value[4..=11];                   // Data = bits 4-11
let data = value.get_bits(4..=11);          // <- above is the same as this

let reserved = value.get_bits(-4);          // Reserved = last 4 bits

if reserved != 0 {
    throw reserved;
}

switch command {
    0 => print(`Data = ${data}`),
    1 => value[4..=11] = data / 2,
    2 => value[3] = !flag,
    _ => print(`Unknown: ${command}`)
}
```
