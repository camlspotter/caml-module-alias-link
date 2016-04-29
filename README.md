# OCaml's module alias link test

## How to see it is working

```shell
$ cd simple
$ omake
$ strings test | grep happy
```

See `happy_x` is linked in the executable but not `happy_y`.

## Points you have to know.

Suppose we have a compilation unit `My`, i.e. `my.ml` with the following code:

```
module M = My_m
```

where `My_m` refers another compilation unit `my_m.ml`, and `my.ml` is compiled with `-no-alias-deps`.  Then the compilation of `M` does not have real dependency over `My_m`.  Linking of `my.ml` does not trigger linking of `my_m.ml` if neither of `My_m` or `My.M` is used.  Thus with `-no-alias-deps`, you can create a big hierarchy of modules keeping unused submodules unlinked in the final executables.  Actually, `my.ml` compiles even without the compilation of `my_m.ml`. (The compiler complains a warning 49, though.)

How does it work?  With the above settings, uses of `My.M` in application codes are actually replaced by `My_m` by the compiler.  Thus the compilation units gathered by the special alias `module M = My_m` with `-no-alias-deps` are accessible as submodules but internally treated as independent toplevel modules.  The OCaml linker handles them as toplevel modules: if they are not used at all, they are dropped from the linkage, and the code size reduces as a result. 

## Pitfalls

`-no-alias-deps` is very subtle to use correctly. Here are some points you have to be careful.

### Dependency analysis

You have to use `-as-map` option of `ocamldep` to get better dependency. This is not available 4.02.1.

### The right hand side must be a compilation unit.

If `my.ml` has `module MN = My_m.N`, it is *not* this special alias, therefore if `My` is linked, `My_m` is always linked together.  A special module alias must have a compilation unit in its rhs.

As we have seen, this linker trick with the special module alias utilizes `.cma` linking to drop unused submodules from executables.  This requres submodules exposed to the global name space: submodules must have unique names, for example by prefixed with the library name like `my_`, to avoid possible collisions with other libraries.  This makes `-for-pack` trick completely obsolete with `-no-alias-deps`.

## OMakefile for `-no-alias-deps`

`macro/OMakefile` provides an use example of `-no-alias-deps` with OMake macros.
