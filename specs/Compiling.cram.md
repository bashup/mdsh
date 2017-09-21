## Compilation

### Command-Line Handling

Our fixture and helper:

    $ mdsh() { $TESTDIR/../mdsh "$@"; }
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
    Usage: */mdsh [ --compile | --eval ] markdownfile [args...] (glob)
    [2]
    $ mdsh --compile
    Usage: */mdsh [ --compile | --eval ] FILENAME... (glob)
    [2]
    $ mdsh --compiler
    */mdsh: unrecognized option: --compiler (glob)
    [2]
    $ mdsh -- --compile t1.md
    */mdsh: line *: --compile: No such file or directory (glob)

Help:

    $ mdsh --help
    Usage: */mdsh [ --compile | --eval ] markdownfile [args...] (glob)
    
    Run and/or compile code blocks from markdownfile(s) to bash.
    Use a filename of `-` to run or compile from stdin.
    
    Options:
      -h, --help                Show this help message and exit
      -c, --compile MDFILE...   Compile MDFILE(s) to bash and output on stdout.
      -E, --eval MDFILE...      Compile, but add a shelldown-support footer line
    
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

    $ source $TESTDIR/../mdsh   # test `mdsh-compile` directly to check subshell functionality
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
    > mdsh-compile-python() { printf 'python -c %q\n' "$(cat)"; }
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

So long as they don't contain single quotes:

    $ mdsh --compile - <<'EOF'
    > ```this is a 'weird $one!'
    > ```
    > EOF


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
