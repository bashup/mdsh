# Embedding Source Files

The `mdsh-embed` function embeds a sourced bash file at the current point in the file, wrapped in a `source` command and heredoc to keep it from thinking it's the main program.

The module must exist, of course:

    $ source "$BASHER_INSTALL_BIN/mdsh"; set +e
    $ mdsh-embed ./f1
    Can't find module ./f1
    [69]

    $ cat >f1 <<'EOF'
    > echo "hello from f1"
    > if [[ $0 != $BASH_SOURCE ]]; then echo "I am being sourced"; fi
    > EOF
    $ source ./f1
    hello from f1
    I am being sourced

    $ mdsh-embed ./f1 >f2
    $ bash f2
    hello from f1
    I am being sourced

But it will be looked for on PATH if the name hasn't got a '/' in it:

    $ mkdir demo
    $ mv f1 demo
    $ PATH=demo:$PATH mdsh-embed f1 >f3
    $ bash f3
    hello from f1
    I am being sourced

And heredoc boundary generation is smart, so it can't be confused by something in the embedded module that looks like a boundary:

    $ echo '# --- EOF f1 ---' >>demo/f1
    $ mdsh-embed demo/f1
    * <<'# --- EOF f1.1 ---' (glob)
    echo "hello from f1"
    if [[ $0 != $BASH_SOURCE ]]; then echo "I am being sourced"; fi
    # --- EOF f1 ---
    # --- EOF f1.1 ---

## OS-Specific Handling

On cygwin and msys2 (aka "git bash"), you can't source from a heredoc via fd 0; you have to use process substitution.  So on msys and cygwin, this program should output /dev/fd/63:

    $ cat >my-source <<'EOF'
    > echo "$BASH_SOURCE"
    > EOF
    $ [[ $BASH_VERSINFO == 3 ]] && echo "/dev/fd/63" || OSTYPE=cygwin eval "$(mdsh-embed ./my-source)"
    /dev/fd/63
    $ [[ $BASH_VERSINFO == 3 ]] && echo "/dev/fd/63" || OSTYPE=msys eval "$(mdsh-embed ./my-source)"
    /dev/fd/63

And on every other OS, it should source via /dev/fd/0:

    $ case $OSTYPE in
    > msys|cygwin) echo /dev/fd/0 ;;   # fake pass on msys/cygwin
    > *) eval "$(mdsh-embed ./my-source)" ;;
    > esac
    /dev/fd/0

Note: bash 3.2 doesn't support sourcing from a process substitution, so on Cygwin and msys a newer version of bash is required.
