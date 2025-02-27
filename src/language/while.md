While Loop
==========

{{#include ../links.md}}

`while` loops follow C syntax.

Like C, `continue` can be used to skip to the next iteration, by-passing all following statements;
`break` can be used to break out of the loop unconditionally.

~~~admonish tip.small "Tip: Disable `while` loops"

`while` loops can be disabled via [`Engine::set_allow_looping`][options].
~~~

```rust
let x = 10;

while x > 0 {
    x -= 1;
    if x < 6 { continue; }  // skip to the next iteration
    print(x);
    if x == 5 { break; }    // break out of while loop
}
```
