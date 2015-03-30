# OCaml 4.02's module alias link test

## How to see it is working

```shell
$ omake
$ strings test | grep happy
```

See `happy_x` is linked in the executable but not `happy_y`.

## Points you have to know.

`module M = My_m` in a compilation unit `my.ml` compiled with `-no-alias-deps` does not create a real dependency over `My_m`.  The code does not make the linking of `My` triggers the linking of `My_m`.  Actually, the code compiles even without the compilation of `My_m`. (The compiler complains a warning 49, though.)

> Problem: `ocamldep` of 4.02.1 does not know `-no-alias-deps` therefore `ocamldep my.ml` prints a false dependency over `my_m.ml`.

With the above settings, any use of `My.M` is replaced by `My_m`.  If `My` and `My_m` are archived into `my.cma`, `My_m` is dropped from an executable using `my.cma`, if neither `My_m` or `My.M` is used.  This is done by the good old behaviour of `.cma` linking.  Thus with `-no-alias-deps`, you can create a big hierarchy of modules keeping unused submodules unlinked.

> Pitfall: If `my.ml` has `module MN = My_m.N`, it is *not* this special alias, therefore if `My` is linked, `My_m` is always linked together.  A special module alias must have a compilation unit in its rhs.

> Problem: as we have seen, this linker trick with the special module alias utilizes `.cma` linking to drop unused submodules from executables.  This requres submodules exposed to the global name space: submodules must have unique names, for example by prefixed with the library name like `my_`, to avoid possible collisions with other libraries.  This makes `-for-pack` trick completely obsolete with `-no-alias-deps`.
