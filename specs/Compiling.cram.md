## Compilation

### Command-Line Handling

Our fixture and helper:

````sh
    $ mdsh() { "$BASHER_INSTALL_BIN/mdsh" "$@"; }
    $ cat >t1.md <<'EOF'
    > ```shell
    > echo yep
    > ```
    > EOF
````

Nominal Success cases:

````sh
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
````

Pipes:

````sh
    $ cat t1.md | mdsh --compile -
    echo yep
    $ cat t1.md | mdsh --compile t1.md - t1.md
    echo yep
    echo yep
    echo yep
    $ cat t1.md | mdsh -
    yep
````

Errors:

````sh
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
    */mdsh: line *: --compile: No such file or directory (glob)
````

Help:

````sh
    $ mdsh --help
    Usage: mdsh [--out FILE] [ --compile | --eval ] markdownfile [args...]
    
    Run and/or compile code blocks from markdownfile(s) to bash.
    Use a filename of `-` to run or compile from stdin.
    
    Options:
      -h, --help                Show this help message and exit
      -c, --compile MDFILE...   Compile MDFILE(s) to bash and output on stdout.
      -E, --eval MDFILE         Compile one file w/a shelldown-support footer line
    
    $ 
````

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

    $ source "$BASHER_INSTALL_BIN/mdsh"   # test `mdsh-compile` directly to check subshell functionality
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

unless you use mdsh-source, instead:

````sh
    $ mdsh-source <<'EOF'
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

    $ type -t mdsh-lang-python || echo [$?]
    function
````

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

````sh
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
````

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

````sh
    $ mdsh --compile - <<'EOF'
    > ```mdsh
    > echo hello world!
    > ```
    > EOF
    hello world!
````

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

````sh
    $ mdsh --compile - <<'EOF'
    > ~~~xyz
    > ```this is a weird $one!
    > blah
    > ```
    > ~~~
    > EOF
````

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

And `shell @mdsh` blocks are treated the same as plain `mdsh` blocks:

````sh
    $ mdsh --compile - <<'EOF'
    > Note: This syntax is deprecated; use 'shell @mdsh' or 'bash @mdsh' instead
    > ```shell mdsh
    > echo "Hello from mdsh!"
    > ```
    > EOF
    Hello from mdsh!

    $ mdsh --compile - <<'EOF'
    > ```shell @mdsh foo bar baz
    > echo "Hello from mdsh!"
    > ```
    > EOF
    Hello from mdsh!
````

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

````shell
    $ ls x || echo fail
    ls: *: No such file or directory (glob)
    fail

    $ mdsh -o x || echo [$?]
    Usage: mdsh [--out FILE] [ --compile | --eval ] markdownfile [args...]
    [64]

    $ ls x || echo fail
    ls: *: No such file or directory (glob)
    fail

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

    $ { mdsh --out x --compile - || echo [$?]; }  <<'EOF'
    > ```mdsh
    > return 27
    > ```
    > EOF
    [27]

````

### Building an Output File with mdsh-make

````shell
    $ set +e
    $ mkdir t; cd t
    $ mdsh-make ../t1.md t1 echo building
    building
    $ cat t1
    echo yep
    $ same-time() { [[ ! "$1" -nt "$2" && ! "$1" -ot "$2" ]]; }
    $ same-time ../t1.md t1; echo $?
    0
    $ mdsh-make ../t1.md t1 echo building
    $ same-time ../t1.md t1; echo $?
    0
    $ touch -t 201101012310.45 ../t1.md
    $ same-time ../t1.md t1; echo $?
    1
    $ mdsh-make ../t1.md t1 echo building
    building
    $ same-time ../t1.md t1; echo $?
    0
````

mdsh-make should also pass through errors, and without marking the output as compiled:

````shell
    $ cat >err.md <<'EOF'
    > ```mdsh
    > return 27
    > ```
    > EOF

    $ mdsh-make err.md err.sh echo testing
    testing
    [27]

    $ mdsh-make err.md err.sh echo testing
    testing
    [27]
````



### Caching with mdsh-cache

Automatic cache dir creation, name flattening, and make:

~~~shell
    $ cd ..
    $ mdsh-cache my-cache t1.md "" echo building
    building
    $ echo "$REPLY"
    my-cache/t1.md

    $ ls my-cache
    t1.md
    $ cat my-cache/t1.md
    echo yep

    $ mdsh-cache my-cache t1.md ./t/../t1.md echo again
    again
    $ echo "$REPLY"
    my-cache/%2E%2Ft%2F..%2Ft1.md
    $ ls my-cache
    %2E%2Ft%2F..%2Ft1.md
    t1.md
    $ cat my-cache/%2E%2Ft%2F..%2Ft1.md
    echo yep
    $ mdsh-cache my-cache t1.md ./t/../t1.md echo again
~~~

### run-markdown and mdsh-run

````sh
# Source
    $ run-markdown t1.md
    yep

    $ cat >t2.md <<'EOF'
    > ```shell
    > printf '%q\n' "$@"
    > ```
    > EOF
    $ run-markdown t2.md x y z
    x
    y
    z

# mdsh-run caches in $MDSH_CACHE with key $2

    $ MDSH_CACHE=cache2 mdsh-run t2.md MYKEY a b c
    a
    b
    c

    $ ls cache2
    MYKEY

    $ cat cache2/MYKEY
    printf '%q\n' "$@"

# mdsh-use-cache sets MDSH_CACHE

    $ mdsh-use-cache cache3
    $ mdsh-run t2.md "" x
    x

    $ ls cache3
    t2.md

    $ declare -p MDSH_CACHE
    declare -- MDSH_CACHE="cache3"

# mdsh-use-cache w/no-args sets to {XDG_CACHE_HOME|HOME/.cache}/mdsh

    $ XDG_CACHE_HOME=/my/cache HOME=/home/me mdsh-use-cache
    $ declare -p MDSH_CACHE
    declare -- MDSH_CACHE="/my/cache/mdsh"

    $ XDG_CACHE_HOME= HOME=/home/me mdsh-use-cache
    $ set +x; declare -p MDSH_CACHE
    declare -- MDSH_CACHE="/home/me/.cache/mdsh"

    $ XDG_CACHE_HOME= HOME= mdsh-use-cache
    $ declare -p MDSH_CACHE
    declare -- MDSH_CACHE=""


# or runs run-markdown if cache disabled

    $ run-markdown() { echo "run-markdown" "$@"; }

    $ MDSH_CACHE= mdsh-run t2.md "" q
    run-markdown t2.md q
````
