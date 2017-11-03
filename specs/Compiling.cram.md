## Compilation

### Command-Line Handling

Our fixture and helper:

    $ ln -s $TESTDIR/../mdsh.md mdsh
    $ mdsh() { ./mdsh "$@"; }
    $ cat >t1.md <<'EOF'
    > ```shell
    > echo yep
    > ```
    > EOF

Nominal Success cases:


    $ mdsh --compile t1.md
    echo yep
    $ mdsh -c t1.md
    echo yep
    $ mdsh --eval t1.md
    echo yep
    __status=$? eval 'return $__status || exit $__status' 2>/dev/null
    $ mdsh -E t1.md
    echo yep
    __status=$? eval 'return $__status || exit $__status' 2>/dev/null
    $ mdsh --compile t1.md t1.md
    echo yep
    echo yep


    $ mdsh t1.md
    yep
    $ mdsh -- t1.md
    yep

Pipes:


    $ cat t1.md | mdsh --compile -
    echo yep
    $ cat t1.md | mdsh --compile t1.md - t1.md
    echo yep
    echo yep
    echo yep
    $ cat t1.md | mdsh -
    yep

Errors:


    $ mdsh
    Usage: mdsh [--out FILE] [ --compile | --eval ] markdownfile [args...]
    [64]
    $ mdsh --compile
    Usage: mdsh --compile FILENAME...
    [64]
    $ mdsh --eval
    Usage: mdsh --eval FILENAME
    [64]
    $ mdsh --eval - </dev/null
    Usage: mdsh --eval FILENAME
    [64]
    $ mdsh --compiler
    mdsh: unrecognized option: --compiler
    [64]
    $ mdsh -- --compile t1.md
    ./mdsh: line *: --compile: No such file or directory (glob)

Help:

    $ mdsh --help
    Usage: mdsh [--out FILE] [ --compile | --eval ] markdownfile [args...]
    
    Run and/or compile code blocks from markdownfile(s) to bash.
    Use a filename of `-` to run or compile from stdin.
    
    Options:
      -h, --help                Show this help message and exit
      -c, --compile MDFILE...   Compile MDFILE(s) to bash and output on stdout.
      -E, --eval MDFILE         Compile one file w/a shelldown-support footer line
    
    $ 

### Basics

Shell code should compile to itself:

    $ mdsh --compile - <<'EOF'
    > ```shell
    > echo hey!
    > ```
    > EOF
    echo hey!

Unrecognized languages append to a variable:

    $ mdsh --compile - <<'EOF'
    > ```python
    > print("hiya")
    > ```
    > EOF
    mdsh_raw_python+=($'print("hiya")\n')

Languages with a runtime function get embedded, and mdsh blocks are executed silently:

    $ source $TESTDIR/../mdsh.md   # test `mdsh-compile` directly to check subshell functionality
    $ mdsh-compile <<'EOF'
    > ```mdsh
    > mdsh-lang-python() { python; }
    > ```
    > ```python
    > print("hiya")
    > ```
    > EOF
    {
        python
    } <<'```'
    print("hiya")
    (```) (re)

...with no side-effects from the mdsh blocks:

    $ type -t mdsh-lang-python || echo [$?]
    [1]

You can define mdsh-compile-X functions to generate code directly:

    $ mdsh --compile - <<'EOF'
    > ```mdsh
    > mdsh-compile-python() { printf 'python -c %q\n' "${1%$'\n'}"; }
    > ```
    > ```python
    > print("hiya")
    > ```
    > EOF
    python -c print\(\"hiya\"\)

And if there's both a lang-X and compile-X function, lang takes precedence:

    $ mdsh --compile - <<'EOF'
    > ```mdsh
    > mdsh-compile-python() { printf 'python -c %q\n' "$(cat)"; }
    > mdsh-lang-python() { python; }
    > ```
    > ```python
    > print("hiya")
    > ```
    > EOF
    {
        python
    } <<'```'
    print("hiya")
    (```) (re)


### Unknown Language Blocks

When `run-markdown` encounters a block with an unknown tag, it should append to a `mdsh_raw_`*tag* variable:

    $ mdsh --compile - <<'EOF'
    > ```foo
    > test
    > this!
    > ```
    > EOF
    mdsh_raw_foo+=($'test\nthis!\n')

And invalid characters for a variable name are replaced with `_`:

    $ mdsh --compile - <<'EOF'
    > ```a weirdly-formatted thing!
    > more
    > ```
    > EOF
    mdsh_raw_a_weirdly_formatted_thing_+=($'more\n')

But known languages are *not* added to a variable:

    $ mdsh --compile - <<'EOF'
    > ```mdsh
    > echo hello world!
    > ```
    > EOF
    hello world!


### Bad Tags

You can use weird language tags:

    $ mdsh --compile - <<'EOF'
    > ```this is a weird $one!
    > blah
    > ```
    > EOF
    mdsh_raw_this_is_a_weird__one_+=($'blah\n')

Even if they contain single quotes:

    $ mdsh --compile - <<'EOF'
    > ```this is a 'weird $one!'
    > ```
    > EOF
    mdsh_raw_this_is_a__weird__one__+=('')

But not if they have no language at all:

    $ mdsh --compile - <<'EOF'
    > ```
    > X
    > ```
    > EOF

Or are contained in a `~~~` block:

    $ mdsh --compile - <<'EOF'
    > ~~~xyz
    > ```this is a weird $one!
    > blah
    > ```
    > ~~~
    > EOF


### Edge Cases

Blocks that run off the end are ok, and blank lines/trailing whitespace are preserved:

    $ mdsh --compile - <<'EOF'
    > ```this is a weird $one!
    > blah
    > 
    > 
    > yada
    >  
    > 
    > 
    > EOF
    mdsh_raw_this_is_a_weird__one_+=($'blah\n\n\nyada\n \n\n\n')

You can define handlers for empty languages, too:

    $ mdsh --compile - <<'EOF'
    > ```mdsh
    > mdsh-compile-() { printf %s "$1"; }
    > ```
    > ```
    > this is a test
    > ```
    > EOF
    this is a test

And `shell mdsh` blocks are treated the same as plain `mdsh` blocks:

    $ mdsh --compile - <<'EOF'
    > ```shell mdsh
    > echo "Hello from mdsh!"
    > ```
    > EOF
    Hello from mdsh!


### Running Compiled Code

Positional args are passed to the compiled code, and `$0 == $BASH_SOURCE`:

    $ mdsh - a "b c" d <<'EOF'
    > ```shell
    > printf "%s\n" "$@"
    > ```
    > EOF
    a
    b c
    d
    $ mdsh - <<'EOF'
    > ```shell
    > [[ $BASH_SOURCE == $0 ]] && echo yep!
    > ```
    > EOF
    yep!

Unfortunately, the only way you can programmatically *make* `$0 == $BASH_SOURCE` is if they're *empty*...  So we also set `$MDSH_ZERO` to what they "should have been"

    $ mdsh - <<'EOF'
    > ```shell
    > echo "\$MDSH_ZERO='$MDSH_ZERO', \$0='$0', \$BASH_SOURCE='$BASH_SOURCE'"
    > ```
    > EOF
    $MDSH_ZERO='-', $0='', $BASH_SOURCE=''

### Compiling to an Output File

Passing `--out *file*` as the first option overwrites that file with the compilation (or run) result, only if it completes without an error:

```shell
    $ ls x || echo [$?]
    ls: cannot access 'x': No such file or directory
    [2]

    $ mdsh -o x || echo [$?]
    Usage: mdsh [--out FILE] [ --compile | --eval ] markdownfile [args...]
    [64]

    $ ls x || echo [$?]
    ls: cannot access 'x': No such file or directory
    [2]

    $ mdsh -o x --compile - <<'EOF'
    > ```shell
    > echo hello world
    > ```
    > EOF
    $ bash ./x
    hello world

    $ { mdsh --out x - || echo [$?]; }  <<'EOF'
    > ```shell
    > echo exiting
    > exit 49
    > ```
    > EOF
    [49]
    $ bash ./x
    hello world

    $ { mdsh --out x - || echo [$?]; }  <<'EOF'
    > ```shell
    > echo echo exiting
    > ```
    > EOF
    $ bash ./x
    exiting
```
