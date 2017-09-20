## Multi-Lingual Literate Programming with `mdsh`

`mdsh` is a script interpreter for markdown files.  By default, it only executes `shell` code blocks as bash code, but you can also use `mdsh` blocks to define handlers for other languages.  For example, this script will run `python`-tagged code blocks by piping them to the `python` command:

~~~markdown
# Hello World in Python

​```mdsh
mdsh-lang-python() { python; }
​```

​```python
print("hello world!")
​```
~~~

If you have [basher](https://github.com/basherpm/basher), you can install `mdsh` by running `basher install bashup/mdsh`.  Otherwise just clone this repo and copy or link the `mdsh` file to a directory on your `PATH`.  (Or just [download the script directly](https://github.com/bashup/mdsh/raw/master/mdsh) to a directory on your `PATH`.)

## Usage

Running `mdsh` *markdownfile args...* will read and translate backquote-fenced code blocks from *markdownfile* into bash script code, based on the language listed on the block and any translation rules you've defined.  The resulting translated script is then run, passing in *args* as positional arguments to the script.

Blocks tagged as `shell` are interpreted as bash code, and directly copied to the translated script.  So arguments passed to `mdsh` after the path to the markdown file are available as `$1`, `$2`, etc. within the top-level code of `shell` blocks, just like in a regular bash script.

(Typically, you won't run `mdsh` directly, but will put `#!/usr/bin/env mdsh` on the first line of your markdown file instead, and make it executable with `chmod +x`.  That way, users of your script won't need to do anything special to run it.)

### Data Blocks

The contents of blocks that are *not* tagged `shell` or `mdsh` are treated as *data* by default: their contents are added to bash arrays named according to the language on the block, e.g.:

~~~markdown
# Data Arrays Example
Blocks without a defined language processor get translated to a variable
assignment like `mdsh_raw_json+=(\#\ block\ 0)` at that point in the
generated script:

​```json
{ "hello": "world" }
​```
​```shell
echo "${mdsh_raw_json[0]}"   # prints '{ "hello": "world" }'
​```
​```json
{ "this is": "great" }
​```
​```shell
echo "${mdsh_raw_json[0]}"   # prints '{ "hello": "world" }'
echo "${mdsh_raw_json[1]}"   # prints '{ "this is": "great" }'
​```

## Naming Rules
Language names are *case sensitive*, and non-identifier
characters in language names become `_` in variable names:

​```C++
// hey
​```
​```shell
echo "${mdsh_raw_C__[0]}"   # prints '// hey'
​```
~~~

Of course, it would be even better if you could automate the processing of these blocks, so you don't have to follow every block with another `shell` block to process it!  Which is why the next section is on...

### Processing Non-`shell` Languages

To automate the handling of non-`shell` language blocks, you can define one or more `mdsh` blocks, containing "hook functions".   `mdsh` blocks are a bit like a Makefile, in that they define *rules* for how to *build* parts of your script, based on the language used.

These build rules are specified by defining specially-named bash functions.  Unlike functions in `shell` blocks, these functions are *not* part of your script and therefore can't be called directly.  Instead, `mdsh` itself invokes them (or copies out their source code), whenever subsequent blocks match the functions' names:

* An `mdsh-lang-X` function is a template for code to be run when a block of language `X` is encountered.  Its function body is copied to the translated script as a bash compound statment (i.e. in curly braces`{...}`) , that will execute with the block contents as a heredoc on its standard input.  (Its standard output is the same as the overall script's.)
* An `mdsh-compile-X` function is invoked *at compile time* with the block contents on standard input, and must output a bash source code translation of the block on its stdout.
* An `mdsh-after-X` function is a template for code to be run *after* a block of language `X` is encountered.  Its function body is copied to the translated script as a block just after the `mdsh-lang-X` body, `mdsh-compile-X` output, or `mdsh_raw_X+=(contents)` statement.  It does *not* receive the block source, so its standard input and output are those of the script itself.

If both an `mdsh-lang-X` and `mdsh-compile-X` function exist, `mdsh-lang-X` takes precedence.  Defining either one also disables the `$mdsh_raw_X` functionality: only untranslatable "data" blocks are added to the arrays.

If there is no `mdsh-lang-X` or `mdsh-compile-X` however, the `mdsh-after-X` function can read the most recent block's contents from `${mdsh_raw_X[-1]}`.  If you don't unset the array, it will keep growing as more blocks of that language are encountered.

Note: these function names are **case sensitive**, so a block tagged with an uppercase `C` will not trigger the same functions as a block tagged with a lowercase `c`, or vice-versa.  Also, note that because `mdsh` blocks are executed at compile time, they do **not** have access to the script's arguments or I/O: all you can do in them is define hook functions.

Finally, please remember that you shouldn't have *any* code in an `mdsh` block other than hook function definitions!  `mdsh` blocks are *not* part of the translated script, they are part of the *translation process*.  So any functions you define in them won't be around when the script actually runs, and any changes you make to variables won't be still around when the actual script execution happens.

### Making Executable Markdown Files

If you want to make a markdown file directly executable, you can `chmod +x` it and give it a shebang line such as `#!/usr/bin/env mdsh`.  This will then let you run `somescript.md args..` without needing to type `mdsh` in front of it.

If you want to get rid of the `.md` extension on your script, you'll probably want to also add a line to tell Github, your editor, etc. that the file is still markdown, e.g.:

```sh
#!/usr/bin/env mdsh
<!-- ex: set syntax=markdown : -->
```

This will tell Github, atom (with the `vim-modeline` package), and other editor/display tools that the file is actually Markdown.

(Alternately, you can keep the `.md` on the file for editing, but use an extensionless symlink to run it without needing to type the `.md`.)

### Advanced Block Compilation Techniques

Once you've gotten used to doing some `mdsh-lang-X` functions, why not try your hand at some `mdsh-compile` ones?

For example, in the `jqmd` project, I originally had some code that looked like this:

```bash
YAML() { JSON "$(echo "$1" | yaml2json -)"; }

mdsh-lang-yaml() { YAML "$(cat)"; }
```

Which works pretty well, except, since the YAML is a constant value, why not convert it to JSON during compilation?  That way, we could eliminate the runtime overhead (if we save and rerun the compiled script):

```bash
mdsh-compile-yaml() { printf 'JSON %q\n' "$(yaml2json -)"; }
```

Notice the difference between the two functions: the `lang` function is a code *template*.  `mdsh` copies its body into your script source, resulting in code that looks like:

~~~bash
{
    YAML "$(cat)"
} <<'```'
... yaml data here ...
​```
~~~

But the `compile` function simply runs `yaml2json` immediately, and then writes out the translated data, like so:

```bash
JSON ...shell-quoted json here...
```

Notice the use of `printf` with `%q` -- this results in the data being properly escaped to work as a command line argument.  (Take care when you do direct code generation to escape such values properly.  When you need to insert variable data into generated code, always use `printf` with a constant string format, with `%q` placeholders for any standalone arguments.)

Notice too, by the way, that `compile` functions get access to the actual block text, which means that you can do any sort of code generation you like.  For example, I could have taken the output of `yaml2json`, and run `jq` over it, then looped over the output and written bash code to set variables based on the result, or generated code for subcommands based on the specification, or maybe even generated an argument parser from it.  There are all sorts of interesting possibilities for these kinds of code generation techniques!

### Excluding Blocks From The Generated Script

If your script has a lot of documentation examples that contain fenced code blocks, you may want to exclude these from being processed or copied to bash variables.  There are two main ways you can do this.

First, you can change the way you indicate certain code blocks.  All of these are ignored by `mdsh` and do not generate any code:

* Code blocks indented with four spaces, instead of fenced
* Code blocks fenced with `~~~X` instead of `` ```X ``
* Code blocks with no language tag, or with a single quote character anywhere in the language tag

Alternately, you can define empty `mdsh-compile-X` functions in an mdsh block, for each language you want to exclude from the compilation.

## Extending `mdsh` or Reusing its Functions

Sourcing `mdsh` from a bash script will define all its functions, but not actually run a program.  This allows you to change how command line arguments are processed, or predefine additional language hooks, teardown hooks, etc.   (You can also just do it to make use of the included markdown processing functions.)

Note that sourcing `mdsh` will set bash to  ["unofficial strict mode"](http://redsymbol.net/articles/unofficial-bash-strict-mode/) , i.e. `-euo pipefail`.  `mdsh` is written with the assumption that these settings are in effect, so changing them may have undesirable results.

### Available Functions

The following functions are available for your use or alteration in scripts sourcing `mdsh`:

* `run-markdown` *mdfile args...* -- execute the specified markdown file with *args* as its positional arguments (`$1`,  `$2`, etc.)
* `mdsh-error` *format args...* --`printf` *format args* to stderr and terminate the process with errorlevel 2.  (A linefeed is added to the format automatically.)
* `mdsh-compile` -- accepts markdown on stdin and outputs bash code on stdout.  The compilation takes place in a subshell, so hook functions defined in the code being compiled do **not** affect the caller's environment.  Hook functions *already* defined in the caller's environment, however, will be used to translate blocks of the relevant languages.
* `markdown-to-shell` *command language_regexes...* -- read markdown from stdin, writing a sourceable shell script to stdout.  Triple-backquoted blocks whose tag matches any of the specified language-regexes are replaced with *command langtag* and a heredoc for the the block body. If no language-regexes are given, all blocks with a language are processed.

  So for example, `markdown-to-shell process python perl` would output a `process python <<` command for each `python` code block, and and a `process perl <<` command for each `perl` code block.  It's up to you to supply a *command* that can take a language-name argument and process data from its stdin.  (Typically, your full command using this would look something like `source <(markdown-to-shell cmd lang... <somefile.md)`, to parse `somefile.md` and execute the result.)

  **Note**: language regexes are in `sed` syntax, so languages like  `c++` don't need the `+` signs escaped. Conversely, if you want to match one or more of something, use `\+`.

## LICENSE

`mdsh` is copyright 2017 PJ Eby, and MIT-licensed as follows:

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT OWNERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

