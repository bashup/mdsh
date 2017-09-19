## Multi-Lingual Literate Programming with `mdsh`

`mdsh` is a script interpreter for markdown files.  By default, it only executes `shell` code blocks as bash code, but you can define various bash functions to extend it to handle other languages...  and these functions can be defined in your script's  `mdsh` blocks.  For example:

~~~markdown
# Hello World in Python

​```mdsh
mdsh-lang-python() { python; }
​```

​```python
print("hello world!")
​```
~~~

If you have [basher](https://github.com/basherpm/basher), you can install `mdsh` by running `basher install bashup/mdsh`.  Otherwise just clone this repo and copy or link the `mdsh` file to a directory on your `PATH`.

### Usage

Running `mdsh some-document.md args...` will read and interpret triple-backquoted code blocks from `some-document.md`, according to the language listed on the block.  Blocks tagged as `mdsh` are interpreted as bash code, and executed immediately, as part of the markdown-to-shell translation process, so they can define mdsh hook functions (like `mdsh-lang-*`) to affect how other code blocks will be processed.

Blocks tagged with `shell`, on the other hand, can contain arbitrary bash code, inlcuding calls to various mdsh API functions described later in this document.

To process blocks in a language `X`, define a function named `mdsh-lang-X` in an `mdsh` block.  It will then be called whenever a code block of  type `X` is encountered, with the contents of the block on the function's standard input.

Tagged code blocks which lack a language processor at the time they are encountered are appended to `$MDSH_TMP/unprocessed.lang`, where `lang` is the language tag on the code block, and `MDSH_TMP` is an automatically-created temporary directory.  (You can then use the accumulated contents of these files at the end of your script.)

Functions named `mdsh-after-X` will be called *after* a code block of  type `X` is processed.  The function does *not* receive the block contents, and is called whether or not a corresponding  `mdsh-lang-X` function exists.  If there is no `mdsh-lang-X`, the `mdsh-after-X` function can read the block from `$MDSH_TMP/unprocessed.X` and then delete the file.  (This can be useful when running language interpreters that can't execute code via stdin, or when you want the code to run with the main script's stdin.)

#### Command-line Arguments

You can pass additional arguments to `md`, after the path to the markdown file.  These additional arguments are available as `$1`, `$2`, etc. within any top-level `shell` blocks in the markdown file.

#### Making Executable Markdown Files

If you want to make a markdown file directly executable, you can `chmod +x` it and give it a shebang line such as `#!/usr/bin/env mdsh`.  This will then let you run `somescript.md args..` without needing to type `mdsh` in front of it.

A possible downside to this approach is that most markdown editors, converters, and viewers will treat the shebang as a level-one heading.  To avoid this, you can put something like this on the first line instead:

```sh
``exec mdsh "$0" "$@"``
```

Most markdown processors will treat the above as a standalone paragraph containing a bit of code, while most shells will interpret it as a POSIX `sh` command to run `mdsh` on the file, passing along any extra arguments.  (Unlike a `#!`  line, though, this won't work with all shells, nor will the file be executable *without* the use of a shell, so consider your expected usage scenarios carefully before using this trick!)

### Extending `mdsh` or Reusing its Functions

Sourcing `mdsh` from a bash script will define all its functions, but not actually run a program.  This allows you to change how command line arguments are processed, or predefine additional language hooks, teardown hooks, etc.   (You can also just do it to make use of the included markdown processing functions.)

Note that sourcing `mdsh` will set bash to  ["unofficial strict mode"](http://redsymbol.net/articles/unofficial-bash-strict-mode/) , i.e. `-euo pipefail`.  These bash modes should be in effect when executing any `mdsh`-defined functions, or Bad Things May Happen.  (For example, trying to write files to your root filesystem!)  It's highly recommended that you leave these modes in place.

#### Available Functions

The following functions are available for your use or alteration, both in markdown `shell` blocks and in scripts sourcing `mdsh`:

* `run-markdown` *mdfile args...* -- execute the specified markdown file with *args* as its positional arguments (`$1`,  `$2`, etc.)
* `mdsh-error` *args...* -- echo *args* to stderr and return failure (which will cause the script to terminate unless you've disabled `-e`).
* `mdsh-create-tempdir` -- Sets `$MDSH_TMP` to a newly-created temporary directory if it isn't already defined and existing.  `mdsh` calls this automatically before writing `unprocessed.lang` files, so you only need to call this if you're trying to write files of your own to `$MDSH_TMP`.  (Note: do not set `$MDSH_TMP` except by calling this function, as the named directory will be deleted on script termination!)
* `mdsh-delete-tempdir` -- Unsets `$MDSH_TMP`, after deleting the named directory and its contents (if it existed).  You can call this function at any time to reset the current temp directory, but it is also automatically called when the overall script is exited.  (See the section below on "Temporary Files and Exit Trap" for more details.)


* `markdown-to-shell` *command language_regexes...* -- read markdown from stdin, writing a sourceable shell script to stdout.  Triple-backquoted blocks whose tag matches any of the specified language-regexes are replaced with *command langtag* and a heredoc for the the block body. If no language-regexes are given, all blocks with a language are processed.

  So for example, `markdown-to-shell process python perl` would output a `process python <<` command for each `python` code block, and and a `process perl <<` command for each `perl` code block.  It's up to you to supply a *command* that can take a language-name argument and process data from its stdin.  (Typically, your full command using this would look something like `source <(markdown-to-shell cmd lang... <somefile.md)`, to parse `somefile.md` and execute the result.)

  **Note**: language regexes are in `sed` syntax, so languages like  `c++` don't need the `+` signs escaped. Conversely, if you want to match one or more of something, use `\+`.

#### Temporary Files and Exit Trap

In order to clean up any temporary files generated by a script, `mdsh` defines an `EXIT` trap when sourced.  The trap function is called `mdsh-teardown`, and by default all it does is run `mdsh-delete-tmpdir`.  You can override it to do additional things, or change the exit trap to do other things and then call `mdsh-teardown`.

(Note that this means that if you source anything that defines a different `EXIT` trap, the temporary directory will not be cleaned up at script exit.)

## LICENSE

`mdsh` is copyright 2017 PJ Eby, and MIT-licensed as follows:

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT OWNERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

