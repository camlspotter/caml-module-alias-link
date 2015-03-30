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

`-no-alias-deps` is very subtle to use correctly. Here are some points you have to be careful.

`ocamldep` of 4.02.1 does not know `-no-alias-deps` therefore `ocamldep my.ml` prints a false dependency over `my_m.ml`.

If `my.ml` has `module MN = My_m.N`, it is *not* this special alias, therefore if `My` is linked, `My_m` is always linked together.  A special module alias must have a compilation unit in its rhs.

As we have seen, this linker trick with the special module alias utilizes `.cma` linking to drop unused submodules from executables.  This requres submodules exposed to the global name space: submodules must have unique names, for example by prefixed with the library name like `my_`, to avoid possible collisions with other libraries.  This makes `-for-pack` trick completely obsolete with `-no-alias-deps`.

### `-open My` trick

There is a trick to use `-no-alias-deps` keeping your programming style with `-for-pack`:

```shell
$ ocamlc -c -open My -o my_m.cmo m.ml
```

* You do not need to prefix your module files with `my_` for example. You can keep using `m.ml`.
* If your "submodules" have some dependencies between them, say, if module `N` is used lie `N.blah` in `m.ml`, you can keep it as is. No need to rename it to `My_n.blah`.
* In `my.ml`, you have to have a special module alias `module M = My_m`, and must be compiled before the compilation of `m.ml`.

With `-open My` and `my.cmi`, the use of `N.blah` in `m.ml` points to `My.N.blah` and this is replaced by `My_n.blah`. The compilation of `My_n` is created by `ocamlc -c -open My -o my_m.cmo m.ml`.

However, this trick also requires care to use correctly:

* ocamldep does not understand the trick at all. If you have `m.ml` and `n.ml` and `N` is used inside `M`, the dependency you have to add is *not* `n.cmo: m.cmo` but `my_n.cmo: my_m.cmo`. This requires modification of your build tools/scripts.
* `ocamlc -c -open My -o m.cmo m.ml` + `mv m.cmo my_m.cmo` seems to be different from `ocamlc -c open My -o my_m.cmo m.ml`. 

