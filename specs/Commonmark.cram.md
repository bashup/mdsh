## Commonmark Fence Parsing

`mdsh`'s parser follows the [Commonmark fenced code blocks](http://spec.commonmark.org/0.28/#fenced-code-blocks) specification; we test this by invoking it with a dummy "compiler" that outputs information about the blocks the parser encounters:

```shell
    $ source "$TESTDIR/../mdsh.md"; set +e
    $ dummy() {
    >     printf "indent='%s' fence='%s' info='%s'\n%s" "$indent" "$fence" "$2" "$3";
    > }
```

### Invalid Openers

An opening code fence:

* has at least three backquotes or tildes
* is indented by no more than three spaces
* doesn't mix characters, and
* doesn't include backquotes in the info string

Therefore, all of these are invalid fence openers and should not be recognized as code blocks:

```shell
    $ mdsh-parse dummy <<<'
    > ~~
    > ~~~~ `
    > ``` `
    >     ````
    > ~`~`
    > '
```

### Info Strings

Leading and trailing whitespace are trimmed from the info string:

```shell
    $ mdsh-parse dummy <<<'
    > ```  some info  
    > text
    > ```'
    indent='' fence='```' info='some info'
    text
```

### Closing Fences

A closing code fence:

* is at least as long as the opening fence
* is indented by no more than three spaces
* uses the same characters as the opening fence
* has nothing but spaces after it

```````shell
    $ mdsh-parse dummy <<<'
    > ~~~test 1
    > ~~ `
    > ``` `
    >     ~~~~
    > ~`~
    > ~~~test
    > ~~~~~test
    > ~~~~ 
    > ````test 2
    > ```
    > ````testing
    > ````'
    indent='' fence='~~~' info='test 1'
    ~~ `
    ``` `
        ~~~~
    ~`~
    ~~~test
    ~~~~~test
    indent='' fence='````' info='test 2'
    ```
    ````testing
```````


### Indentation

Indentation up to the same amount as the fence is removed from the block contents, and the closing indent doesn't have to match the opening indent:

```shell
    $ mdsh-parse dummy <<<'
    >  ```
    >   block 1, indent 2
    >  block 1, indent 1
    > block 1, indent 0
    >    ```
    >   ```
    > block 2, indent 0
    >  block 2, indent 1
    >   block 2, indent 2
    >    block 2, indent 3
    >  ```
    >    ```
    > block 3, indent 0
    >  block 3, indent 1
    >   block 3, indent 2
    >    block 3, indent 3
    >     block 3, indent 4
    > ```'
    indent=' ' fence='```' info=''
     block 1, indent 2
    block 1, indent 1
    block 1, indent 0
    indent='  ' fence='```' info=''
    block 2, indent 0
    block 2, indent 1
    block 2, indent 2
     block 2, indent 3
    indent='   ' fence='```' info=''
    block 3, indent 0
    block 3, indent 1
    block 3, indent 2
    block 3, indent 3
     block 3, indent 4
```

### Run-Off
A block is allowed to run off the end of the file:
```shell
    $ mdsh-parse dummy <<<'
    > ```test
    > text'
    indent='' fence='```' info='test'
    text
```
