## Extending mdsh

It should be possible to extend mdsh by sourcing it:

    $ cat >foosh <<'EOF'
    > #!/usr/bin/env bash
    > source "$BASHER_INSTALL_BIN/mdsh"
    > mdsh:file-header() { echo 'echo "# header"'; }
    > mdsh:file-footer() { echo 'echo "# footer"'; }
    > if [[ $0 == $BASH_SOURCE ]]; then mdsh-main "$@"; fi
    > EOF
    $ chmod +x foosh

And the result should give error/help messages under its new name:

    $ ./foosh
    Usage: foosh [--out FILE] [ --compile | --eval ] markdownfile [args...]
    [64]

And the extended program's mdsh:file-header and mdsh:file-footer will wrap anything done with it:

    $ cat >t1.md <<<$'```shell\necho t1.md\n```'

    $ ./foosh t1.md
    # header
    t1.md
    # footer

With the header and footer appearing before and after the *entire* compiled file list:

    $ ./foosh --compile <(echo $'```shell\n# file 1 here\n```') t1.md <(echo $'```shell\n# file 3 here\n```')
    echo "# header"
    # file 1 here
    echo t1.md
    # file 3 here
    echo "# footer"

But *before* the eval footer:

    $ ./foosh --eval t1.md
    echo "# header"
    echo t1.md
    echo "# footer"
    __status=$? eval 'return $__status || exit $__status' 2>/dev/null
