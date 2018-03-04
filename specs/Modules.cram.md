## Modules

The `mdsh-module` function should run its arguments on first use, but not subsequent uses with the same module name:

~~~shell
    $ source "$TESTDIR/../mdsh.md"; set +e
    $ mdsh-module foo echo "first!"
    first!
    $ mdsh-module foo echo "first!"
    $ mdsh-module bar echo "second"
    second
~~~

While executing, the command should see a local value of `MDSH_MODULE` equal to the module name:

~~~shell
    $ load() { echo "$MDSH_MODULE"; }
    $ mdsh-module baz load
    baz
~~~

This value should *not* be exported to other processes:

~~~shell
    $ mdsh-module spam bash -c 'echo ${MDSH_MODULE-safe!}'
    safe!
~~~

### "Main" Blocks

The compilation system uses `MDSH_MODULE` to decide whether to execute `shell main` and `mdsh main` blocks, e.g. when these blocks are compiled:

```shell main
#!/usr/bin/env bash
# This file header is only included when built as a main program
```

```shell
# This line is always here
```

```shell mdsh
echo "# And so is this"
```

```shell mdsh main
echo "# This footer is only in the main program"
```

The result is different depending on whether it's done as a module or not:

~~~shell
    $ mdsh-module zebra mdsh-source "$TESTDIR/$TESTFILE"
    # This line is always here
    # And so is this

    $ mdsh-source "$TESTDIR/$TESTFILE"
    #!/usr/bin/env bash
    # This file header is only included when built as a main program
    # This line is always here
    # And so is this
    # This footer is only in the main program
~~~

