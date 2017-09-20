## Compilation

    $ source $TESTDIR/../mdsh

### Basics

Shell code should compile to itself:

    $ mdsh-compile <<'EOF'
    > ```shell
    > echo hey!
    > ```
    > EOF
    echo hey!

Unrecognized languages append to a variable:

    $ mdsh-compile <<'EOF'
    > ```python
    > print("hiya")
    > ```
    > EOF
    mdsh_raw_python+=($'print("hiya")\n')

Languages with a runtime function get embedded, and mdsh blocks are executed silently:

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

    $ mdsh-compile <<'EOF'
    > ```mdsh
    > mdsh-compile-python() { printf 'python -c %q\n' "$(cat)"; }
    > ```
    > ```python
    > print("hiya")
    > ```
    > EOF
    python -c print\(\"hiya\"\)

And if there's both a lang-X and compile-X function, lang takes precedence:

    $ mdsh-compile <<'EOF'
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

    $ __COMPILE__ foo <<'EOF'
    > test
    > this!
    > EOF
    mdsh_raw_foo+=($'test\nthis!\n')

And invalid characters for a variable name are replaced with `_`:

    $ __COMPILE__ "a weirdly-formatted thing!" <<'EOF'
    > more
    > EOF
    mdsh_raw_a_weirdly_formatted_thing_+=($'more\n')

But known languages are *not* added to a variable:

    $ __COMPILE__ mdsh <<'EOF'
    > echo hello world!
    > EOF
    hello world!


### Bad Tags

You can use weird language tags:

    $ markdown-to-shell TEST <<'EOF'
    > ```this is a weird $one!
    > ```
    > EOF
    TEST <<'```' 'this is a weird $one!'
    (```) (re)

So long as they don't contain single quotes:

    $ markdown-to-shell TEST <<'EOF'
    > ```this is a 'weird $one!'
    > ```
    > EOF


### Running Compiled Code

Positional args are passed to the compiled code:

    $ run-markdown <(cat <<'EOF'
    > ```shell
    > printf "%s\n" "$@"
    > ```
    > EOF
    > ) a "b c" d
    a
    b c
    d
