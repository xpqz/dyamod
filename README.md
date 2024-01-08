# WIBNI Dyalog had a package and module concept?

Dyalog has no concept of a `package` or `module` often found in other languages. This is one source of confusion in `tatin`, which tries to implement one without the other. In the below, I've borrowed the [modules system from Python](https://docs.python.org/3/reference/simple_stmts.html#the-import-statement), as a thought experiment, and tried to superimpose that onto Dyalog APL, to see what that might look like. I've chosen Python as the model, as it's the one hammer I'm most comfortable with, but there are several, different, approaches to software building blocks. 

> Why do we need packages and modules when we've managed *just fine* for the last 50 years without?

The reason is that with P&M, the users have more flexibility in how they wish to organise their source code into files. Even if you don't believe that's necessary, `tatin` desperately needs this.

> I'm perfectly happy with [Link](https://dyalog.github.io/link), thank you.

`Link` solves a different problem: somewhat simplified; serialising, and deserialising the contents of a workspace to and from files that can be versioned in something like `git`. `Link` does *not* give you the freedom to organise code as you choose: for example, you can't have a single file with all your definitions in when using Link. Link does a good job solving the problem it set out to solve, but it's a different problem we're discussing here. 

> Why should we trust the user to organise their code? [joke]

There are some suspicions against "the script model" where definitions and executions and data are seemingly mixed up and expected to run from top to bottom. Whilst this *is* a problem in today's Dyalog, meaning that fixing `.apls` files "willy nilly" would lead to chaos and confusion, *this* is one of the problems that having a language level package and module system would solve. 

> Meh. All you're doing is making namespaces from filenames and recusively loading; not very novel...

Basically, yes (plus ensuring everything is only ever loaded once). The whole point is that it's not meant to be novel. The other team has been doing this for decades.

> What does it *not* do?

A lot of functionality of Python's modules system has been deliberately left out of this model:

1. Lifting subsets directly into the caller's namespace:
    ```python
    import math
    # vs
    from math import prod
    ```
    The `dyamod` model only caters for the former, i.e corresponding to Python's
    ```python
    math.prod([1, 2, 3, 4, 5])
    ```
2. Import aliasing: Python gives you control of the name of an imported package for brevity and collision avoidance:
    ```python
    import foo.bar as foobar
    ```

and much more. See the docs for Python's [import](https://docs.python.org/3/reference/simple_stmts.html#the-import-statement)

## Definitions

`module` -- a file. A `module` is a file which defines a set of names that has values. Equivalent of a scripted namespace (but without the `:namespace`). When a module `mymod.aplm` is imported into APL code elsewhere, it ends up in a namespace called `mymod`. A module can run code, import other modules (or packages, see below), define functions, classes etc. Ideally, as Python can, Dyalog would have the ability to distinguish between a module being used as a module (i.e. imported), or as a program ("script"), which is a handy way to package tests with your modules. 

`package` -- a directory. A `package` is a directory which contains zero or more modules and packages, and in Python's case, an optional (since 3.3) source file with a very specific name, `__init__.py`. The `package` is the higher-order code organisation unit. Importing a `package` also creates a namespace based on the name. Any nested packages or modules become nested namespaces when imported. 

`import caching` -- imported modules are guaranteed to be singletons. This means that no matter how many times you import a module, in different files or from different modules, Dyalog will use the same namespace created the first time the module is imported. Python uses the system dictionary `sys.modules` to keep track of this.

`import search path` -- when Dyalog encounters an `⎕IMPORT` statement (or whatever it is called), it will look for the package or module in a specific set of locations. Python manages this with a system vector called `sys.path`, which should get pre-populated with your Python installation's ideas of where to look for stuff. It can be manipulated programmatically, although this is generally thought of as fragile. Importantly, the current directory is always included. In Python's case, on start-up, the contents of the environment variable `PYTHONPATH` will be added to `sys.path`.

I've invented the `.aplm` extension for this module concept. The proliferation of Dyalog extension is a terrible blight, and this is adding to it -- but it was the simplest way to transparently keep these away from Link.

## Initialisation

When the `dyamod` system is initialised, it creates a namespace `#.sys`:

```apl
∇ Run
  'sys' #.⎕NS⍬          
  'modules' #.sys.⎕NS⍬
  #.sys.(path owner resolve) ← (,⊂,'.') (,⊂,'#') ⍬
∇
```
The components of the `sys` namespace have the following jobs:
- `sys.modules`: loaded module cache
- `sys.path`: module directory search paths
- `sys.owner`: internal scratch variable for recursive import resolution
- `sys.resolve`: internal scratch variable for recursive import resolution

The first time a module is imported, it is `2⎕FIX`ed as a named namespace into `#.sys.modules`. A reference to this is then created in the "owning" namespace. Any subsequent imports of the same module elsewhere will simply generate references to the entry in `#.sys.modules`.

Consider the following code layout:

```
utils
├── series.aplm
├── funs.aplm
└── minmax.aplm
```

Here, we have a single package, `utils`, comprising of three modules: `series`, `funs` and `minmax`. If we look at the contents of those,

`series.aplm`:
```apl
fib ← {
    s÷⍨(p*⍵)-⍵*⍨1-p←2÷¯1+s←5*÷2
}
```

`funs.aplm`:
```apl
#.import 'minmax'

sum ← {
    ⍺+⍵
}

diff ← {
    ⍺-⍵
}

mm ← {
    (minmax.min,minmax.max)⍵
}
```

`minmax.aplm`:
```apl
min ← {
    ⌊/⍵
}

max ← {
    ⌈/⍵
}
```

We can see that the modules `series` and `minmax` are leaves -- they have no module dependencies. The module `funs`, however, depends on `minmax`. When we execute `import 'utils'` at the top level, we want to achieve the following:

1. `sys.modules` should contain the namespaces `series`, `funs`, `minmax` and `utils`.
2. `sys.modules.utils` should have references to `series`, `funs` and `minmax`.
3. `sys.modules.funs` should have a reference to `sys.modules.minmax`.
4. `#` should have a reference to `sys.modules.utils`.

We should now be able to execute

```apl
      utils.series.fib 20
6765
```

Let's show how this works in practice:

```apl
Dyalog APL/S-64 Version 19.0.47997
Serial number: UNREGISTERED - not for commercial use
Fri Jan  5 15:15:58 2024
      ]cd work
/Users/stefan
      ]create # dyamod                      ⍝ Load the modules & packages system
Linked: # → /Users/stefan/work/dyamod
      Run                                   ⍝ Initialise system
      
      ]map sys                              ⍝ Show the layout of the modules system before
#.sys
·   ~ owner path resolve
·   modules
      
      ]cd dyamod
/Users/stefan/work

      import 'utils'                        ⍝ Import the test package
      ]map #                                ⍝ Show the system layout post-import
#                                                
·   ∇ Run cleanpath import module package resolve
·   sys                                          
·   ·   ~ owner path resolve                     
·   ·   modules                                  
·   ·   ·   funs                                 
·   ·   ·   ·   ∇ diff mm sum                    
·   ·   ·   ·   minmax → #.sys.modules.minmax    
·   ·   ·   minmax                               
·   ·   ·   ·   ∇ max min                        
·   ·   ·   series                               
·   ·   ·   ·   ∇ fib                            
·   ·   ·   utils                                
·   ·   ·   ·   funs → #.sys.modules.funs        
·   ·   ·   ·   minmax → #.sys.modules.minmax    
·   ·   ·   ·   series → #.sys.modules.series    
·   utils → #.sys.modules.utils 

      utils.series.fib 20                    ⍝ Run 'fib' from 'utils/series' module
6765
```