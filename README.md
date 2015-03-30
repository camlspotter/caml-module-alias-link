# OCaml 4.02's module alias link test

## How to see it is working

```shell
$ omake
$ strings test | grep happy
```

See `happy_x` is linked in the executable but not `happy_y`.

## Points you have to know.

Suppose we have a compilation unit `My`, i.e. `my.ml` with the following code:

```
module M = My_m
```

where `My_m` refers another compilation unit `my_m.ml`, and `my.ml` is compiled with `-no-alias-deps`.  Then the compilation of `M` does not have real dependency over `My_m`.  Linking of `my.ml` does not trigger linking of `my_m.ml` if neither of `My_m` or `My.M` is used.  Thus with `-no-alias-deps`, you can create a big hierarchy of modules keeping unused submodules unlinked.  Actually, `my.ml` compiles even without the compilation of `my_m.ml`. (The compiler complains a warning 49, though.)

How does it work?  With the above settings, any use of `My.M` is actually replaced by `My_m` by the compiler.  Thus the compilation units gathered by the special alias `module M = My_m` with `-no-alias-deps` are accessible as submodules but internally treated as independent toplevel modules.  The OCaml linker handles them as toplevel modules: if they are not used at fall, they are dropped from the linkage, and the code size reduces as a result. 

### Pitfalls

`ocamldep` of 4.02.1 does not know `-no-alias-deps` therefore `ocamldep my.ml` prints a false dependency over `my_m.ml`.

If `my.ml` has `module MN = My_m.N`, it is *not* this special alias, therefore if `My` is linked, `My_m` is always linked together.  A special module alias must have a compilation unit in its rhs.

As we have seen, this linker trick with the special module alias utilizes `.cma` linking to drop unused submodules from executables.  This requres submodules exposed to the global name space: submodules must have unique names, for example by prefixed with the library name like `my_`, to avoid possible collisions with other libraries.  This makes `-for-pack` trick completely obsolete with `-no-alias-deps`.
