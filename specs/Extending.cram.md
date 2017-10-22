## Extending mdsh

    $ cat >foosh <<'EOF'
    > #!/usr/bin/env bash
    > source ./mdsh
    > mdsh:file-header() { echo "# header"; }
    > mdsh:file-footer() { echo "# footer"; }
    > if [[ $0 == $BASH_SOURCE ]]; then mdsh-main "$@"; fi
    > EOF


    $ chmod +x foosh
    $ ln -s "$TESTDIR/../mdsh.md" mdsh
    $ ./foosh
    Usage: ./foosh [ --compile | --eval ] markdownfile [args...]
    [64]


    $ cat >t1.md <<'EOF'
    > ```shell
    > echo yep
    > ```
    > EOF


    $ ./foosh --compile t1.md
    # header
    echo yep
    # footer


    $ ./foosh -c t1.md
    # header
    echo yep
    # footer


    $ ./foosh --eval t1.md
    # header
    echo yep
    # footer
    __status=$? eval 'return $__status || exit $__status' 2>/dev/null

