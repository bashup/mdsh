# mdsh: a self-hosted Markdown-to-Shell Compiler

``mdsh`` is a compiler and interpreter for literate programs written in markdown and bash, that is itself written as a literate program in markdown and bash.  In the compiled distribution, it contains a LICENSE header (see [LICENSE](LICENSE) file for the terms that apply to this source file as well as the compiled version):

```shell mdsh
@module mdsh.md
@main mdsh-main

@require pjeby/license @comment LICENSE
```

And it expects to run in "bash strict mode":

```shell
set -euo pipefail  # Strict mode
```

**IMPORTANT**: just because a function is named `mdsh-something` and explained in this document does not make it a published API function!  If it's not documented in the README, consider it a private implementation detail

### Contents

<!-- toc -->

- [Parser](#parser)
- [Compiler](#compiler)
  * [API](#api)
    + [mdsh-source](#mdsh-source)
    + [mdsh-compile](#mdsh-compile)
  * [Code Block Handling](#code-block-handling)
    + [fn-exists](#fn-exists)
    + [mdsh-rewrite](#mdsh-rewrite)
  * [Default Languages and Data Handling](#default-languages-and-data-handling)
- [Command-Line Interface](#command-line-interface)
  * [Interpreting Files](#interpreting-files)
  * [--compile (-c)](#--compile--c)
  * [--eval (-E)](#--eval--e)
  * [--out (-o) *file*](#--out--o-file)
  * [Usage Errors](#usage-errors)
  * [Help (-h and --help)](#help--h-and---help)
- [Modules](#modules)
  * [@require](#require)
  * [@module](#module)
  * [@main](#main)
  * [@comment](#comment)
- [Utility Functions](#utility-functions)
  * [mdsh-embed](#mdsh-embed)
  * [mdsh-make](#mdsh-make)
  * [run-markdown](#run-markdown)
- [Startup](#startup)

<!-- tocstop -->

## Parser

`mdsh-parse` is an "event-driven" mini-parser for markdown.  It takes a function name as an argument and a markdown string on stdin, and invokes the function with arguments representing "events".  Currently the only event is `fenced`, which has two extra arguments: the language of the code block, and its contents.

The `$fence` and `$indent` variables can be inspected to tell what the block's original fence string (e.g. `~~~`) was, and how deeply it was indented.  The parser implements the full [Commonmark specification for fenced code blocks](http://spec.commonmark.org/0.28/#fenced-code-blocks), including indentation-removal, fence lengths, allowed characters, etc.:

```shell
mdsh-parse() {
	local cmd=$1 lno=0 block_start lang block ln indent fence close_fence indent_remove
	local open_fence='^( {0,3})(~~~+|```+) *([^`]*)$'
	while let lno++; IFS= read -r ln; do
		if [[ $ln =~ $open_fence ]]; then
			indent=${BASH_REMATCH[1]} fence=${BASH_REMATCH[2]} lang=${BASH_REMATCH[3]} block=
			block_start=$lno close_fence="^( {0,3})$fence+ *\$" indent_remove="^${indent// / ?}"
			while let lno++; IFS= read -r ln && ! [[ $ln =~ $close_fence ]]; do
				! [[ $ln =~ $indent_remove ]] || ln=${ln#$BASH_REMATCH}; block+=$ln$'\n'
			done
			lang="${lang%"${lang##*[![:space:]]}"}"; $cmd fenced "$lang" "$block";
		fi
	done
}
```

## Compiler

### API

#### mdsh-source

`mdsh-source` compiles text from standard input, or from the file specified by `$1` (if given).  It runs in the current shell, so the input is treated as if it were contained in the same file as is currently being compiled (if any.)  `MDSH_SOURCE` is set to `$1` (unless it's `-` or empty).

```shell
mdsh-source() {
	local MDSH_FOOTER= MDSH_SOURCE
	if [[ ${1:--} != '-' ]]; then
		MDSH_SOURCE="$1"
		mdsh-parse __COMPILE__ <"$1"
	else mdsh-parse __COMPILE__
	fi
	${MDSH_FOOTER:+ printf %s "$MDSH_FOOTER"}; MDSH_FOOTER=
}
```

(The actual "compilation" consists simply of parsing the file contents using the `__COMPILE__` function as the event handler.)

#### mdsh-compile

`mdsh-compile` is the same as `mdsh-source`, except it's run in a subshell so that variable and function definitions from `mdsh` blocks in the supplied code can't affect the main process state.

```shell
mdsh-compile() (  # <-- force subshell to prevent escape of compile-time state
	mdsh-source "$@"
)
```

### Code Block Handling

`__COMPILE__` emits bash code for unindented, triple-backquote-fenced code blocks, by interpreting them as command blocks or looking up hook functions, and either copying out their source code (using `mdsh-rewrite`), or invoking them directly:

```shell
__COMPILE__() {
	[[ $1 == fenced && $fence == '```' && ! $indent ]] || return 0  # only unindented ``` code
	local lang="${2//[^_[:alnum:]]/_}"; # convert language to safe variable/function name
	local tag_words=($2);  # check for command blocks first
	if [[ ${tag_words[1]-} == '!'* ]]; then
		set -- "$3" "$2" "$block_start"; eval "${2#*!}"; return
	elif [[ ${tag_words[1]-} == '|'* ]]; then
		echo "${2#*|} <<'\`\`\`'"; printf '%s```\n' "$3"; return
	elif fn-exists mdsh-lang-$lang; then
		mdsh-rewrite mdsh-lang-$lang "{" "} <<'\`\`\`'"; printf '%s```\n' "$3"
	elif fn-exists mdsh-compile-$lang; then
		mdsh-compile-$lang "$3" "$2" "$block_start"
	else
		mdsh-misc "$2" "$3"
	fi

	if fn-exists mdsh-after-$lang; then
		mdsh-rewrite mdsh-after-$lang
	fi
}
```

To do that, it needs to be able to detect what functions exist, and to extract their source code.

#### fn-exists

```shell
# fn-exists: succeed if argument is a function
fn-exists() { declare -F -- "$1"; } >/dev/null
```

#### mdsh-rewrite

```shell
# Output body of func $1, optionally replacing the opening/closing { and } with $2 and $3
mdsh-rewrite() {
	declare -f $1 | sed -e '1d; 2s/^{ $/'"${2-"{"}"'/; $s/^}$/'"${3-"\}"}"'/'
}
```

### Default Languages and Data Handling

By default, `mdsh` supports only `mdsh` and `shell` blocks, with everything else handled as data.  Blocks with no language are ignored, and `mdsh main` and `shell main` blocks are only compiled if a module load is not in progress (i.e., the current file is a "main" file):

```shell
mdsh-misc()          { mdsh-data "$@"; }    # Treat unknown languages as data
mdsh-compile-()      { :; }                 # Ignore language-less blocks

mdsh-compile-mdsh()  { eval "$1"; }         # Execute `mdsh` blocks in-line
mdsh-compile-mdsh_main() { ! @is-main || eval "$1"; }

mdsh-compile-shell() { printf '%s' "$1"; }  # Copy `shell` blocks to the output
mdsh-compile-shell_main() { ! @is-main || printf '%s' "$1"; }
```

Data blocks are processed by emitting code to add the block contents to an `mdsh_raw_X` variable:

```shell
mdsh-data() {
	printf "mdsh_raw_${1//[^_[:alnum:]]/_}+=(%q)\n" "$2"
}
```

And for syntax highlighting convenience, `shell mdsh` blocks and `shell mdsh main` blocks are treated as `mdsh` and `mdsh main` blocks, respectively:

```shell
mdsh-compile-shell_mdsh() {
	indent= fence='```' __COMPILE__ fenced mdsh "$1"
}
mdsh-compile-shell_mdsh_main() {
	indent= fence='```' __COMPILE__ fenced "mdsh main" "$1"
}
```

## Command-Line Interface

The command line interface looks for functions named `mdsh.OPT` to process options of a matching name, and gives a usage error if such an option isn't found.  An option of `--` means "run the file that follows", while a `-` by itself is interpreted as "interpret standard input".  A non-option argument is treated as a file to interpret.  Multiple short options are processed by splitting them into single options.

```shell
# Main program: check for arguments and run markdown script
mdsh-main() {
	(($#)) || mdsh-error "Usage: %s [--out FILE] [ --compile | --eval ] markdownfile [args...]" "${0##*/}"
	case "$1" in
	--) mdsh-interpret "${@:2}" ;;
	--*|-?) fn-exists "mdsh.$1" || mdsh-error "%s: unrecognized option: %s" "${0##*/}" "$1"
		"mdsh.$1" "${@:2}"
		;;
	-??*) mdsh-main "${1::2}" "-${1:2}" "${@:2}" ;;  # split '-abc' into '-a -bc' and recurse
	*)  mdsh-interpret "$@" ;;
	esac
}
```

### Interpreting Files

As described in the documentation, markdown files are run with an empty  `$0` and `$BASH_SOURCE`, but with `$MDSH_ZERO` set to the source file.  This is done by exec-ing bash with an immediate command to eval the compiled version of the source file.

```shell
# Run markdown file as main program, with $0 == $BASH_SOURCE == "" and
# MDSH_ZERO pointing to the original $0.

function mdsh-interpret() {
	printf -v cmd 'eval "$(%q --compile %q)"' "$0" "$1"
	MDSH_ZERO="$1" exec bash -c "$cmd" "" "${@:2}"
}
```

### --compile (-c)

Compile one or more files, appending the results to stdout.  `mdsh` blocks in each file will see an `$MDSH_SOURCE` that points to their filename (unless it's a `-`).

```shell
mdsh.--compile() {
	(($#)) || mdsh-error "Usage: %s --compile FILENAME..." "${0##*/}"
	! fn-exists mdsh:file-header || mdsh:file-header
	for REPLY; do mdsh-compile "$REPLY"; done
	! fn-exists mdsh:file-footer || mdsh:file-footer
}

mdsh.-c() { mdsh.--compile "$@"; }
```

### --eval (-E)

Compile one file, which *cannot* be stdin.  Adds a suffix to ensure the compiled code returns, allowing use in shelldown headers of files that are source-able.

```shell
mdsh.--eval() {
	(($# == 1)) && [[ $1 != - ]] ||
		mdsh-error "Usage: %s --eval FILENAME" "${0##*/}"
	mdsh.--compile "$1"
	echo $'__status=$? eval \'return $__status || exit $__status\' 2>/dev/null'
}

mdsh.-E() { mdsh.--eval "$@"; }
```

### --out (-o) *file*

Send output to the named file, overwriting it in place if and only if the compilation succeeds without error.

```shell
mdsh.--out() {
	if REPLY=("$(mdsh-main "${@:2}")"); then
		exec echo "$REPLY" >"$1"   # handle self-compiling properly
	else exit $?;
	fi
}

mdsh.-o() { mdsh.--out "$@"; }
```

### Usage Errors

```shell
# mdsh-error: printf args to stderr and exit w/EX_USAGE (code 64)
mdsh-error() { printf "$1\n" "${@:2}" >&2; exit 64; }
```

### Help (-h and --help)

```shell
mdsh.--help() {
	printf "Usage: %s [--out FILE] [ --compile | --eval ] markdownfile [args...]\n" "${0##*/}"
	echo -e '
Run and/or compile code blocks from markdownfile(s) to bash.
Use a filename of `-` to run or compile from stdin.

Options:
  -h, --help                Show this help message and exit
  -c, --compile MDFILE...   Compile MDFILE(s) to bash and output on stdout.
  -E, --eval MDFILE         Compile one file w/a shelldown-support footer line\n'
}

mdsh.-h() { mdsh.--help "$@"; }
```

## Modules

### @require

`@require` *module cmd [args...]* will run *cmd args...* if it is the first call to `@require` for the named *module*.  The *cmd* should usually be `mdsh-source` or `mdsh-embed`, to include markdown modules or shell script modules, respectively, but it can be any command or function.  During execution of *cmd*, `MDSH_MODULE` will be set to *modulename*, allowing the running code to know it's being used as a module, potentially compiling itself differently.  (Normally, `MDSH_MODULE` is empty.)

```shell
MDSH_LOADED_MODULES=
MDSH_MODULE=

@require() {
	flatname "$1"
	if ! [[ $MDSH_LOADED_MODULES == *"<$REPLY>"* ]]; then
		MDSH_LOADED_MODULES+="<$REPLY>"; local MDSH_MODULE=$1
		"${@:2}"
	fi
}
```

### @is-main

`@is-main` returns truth if the code being built is a program's "main" module, and false if there is a currently-executing `@require`.

```shell
@is-main() { ! [[ $MDSH_MODULE ]]; }
```

### @module

Output a shebang header and an "automatically-generated from" notice, if building a top-level module.  If `$1` is given, use it to determine the filename in the notice.

```shell
@module() {
	@is-main || return 0
	set -- "${1:-${MDSH_SOURCE-}}"
	echo "#!/usr/bin/env bash"
	echo "# ---"
	echo "# This file is automatically generated from ${1##*/} - DO NOT EDIT"
	echo "# ---"
	echo
}
```

### @main

Output code to run `$1` as the main function, if building a top-level module.

```shell
@main() {
	@is-main || return 0
	MDSH_FOOTER=$'if [[ $0 == "${BASH_SOURCE-}" ]]; then '"$1"$' "$@"; fi\n'
}
```

### @comment

Output the specified file(s) as shell-commented lines, followed by a single blank line.  If `MDSH_SOURCE` is defined (as when compiling from the command line), paths are interpreted relative to its directory (*without* taking symlinks into account).

```shell
@comment() (  # subshell for cd
	! [[ "${MDSH_SOURCE-}" == */* ]] || cd "${MDSH_SOURCE%/*}"
	sed -e 's/^\(.\)/# \1/; s/^$/#/;' "$@"
	echo
)
```

## Utility Functions

### mdsh-embed

Embed a bash module's source in such a way that it believes itself to be `source`d when executed.

```shell
mdsh-embed() {
	local f=$1 base=${1##*/}; local boundary="# --- EOF $base ---" contents ctr=
	[[ $f == */* && -f $f ]] || f=$(command -v "$f") || {
		echo "Can't find module $1" >&2; return 69  # EX_UNAVAILABLE
	}
	contents=$'\n'$(<"$f")$'\n'
	while [[ $contents == *$'\n'"$boundary"$'\n'* ]]; do
		let ctr++; boundary="# --- EOF $base.$ctr ---"
	done
	printf $'{ if [[ $OSTYPE != cygwin && $OSTYPE != msys && -e /dev/fd/0 ]]; then source /dev/fd/0; else source <(cat); fi; } <<\'%s\'%s%s\n' "$boundary" "$contents" "$boundary"
}
```

### mdsh-make

Compile file `$1` to file `$2` if the destination doesn't exist or doesn't have the same timestamp.  Compilation happens in a subshell, and any additional arguments are treated as a command to be run inside the subshell before the compilation.  (They're only run if compilation is about to occur.)  If compilation happens and is successful, `$2` is touched to the same timestamp as `$1`.

```shell
fstamp() {
	if ! stat -c %y "$1" 2>/dev/null; then
		fstamp() { stat -f %m "$1"; }
		fstamp "$1";
	fi
} 2>/dev/null

mdsh-make() {
	[[ -f "$1" && -f "$2" && "$(fstamp "$1")" == "$(fstamp "$2")" ]] || {
		( "${@:3}" && mdsh-main --out "$2" --compile "$1" ) && touch -r "$1" "$2"
	}
}
```

### mdsh-cache

Using cache directory `$1`, and cache key `$3` to generate a filename, run `mdsh-make` to build source file `$2`.  Any additional arguments are passed to `mdsh-make` as a pre-compile command.  The actual cache file name is an escaped form of `$3` (to remove `/` and other special characters).  If empty or not given, `$3` defaults to `$2`.  The cache directory is created if it does not exist.

```shell
mdsh-cache() {
	[[ -d "$1" ]] || mkdir -p "$1"
	flatname "${3:-$2}"; REPLY="$1/$REPLY"; mdsh-make "$2" "$REPLY" "${@:4}"
}
```

### flatname

Escape `$1` to use as a "flat" filename with no `/`, `<`, or `>` characters and no leading `.`, returning it in `$REPLY`.

```shell
flatname() {
	REPLY="${1//\%/%25}"; REPLY="${REPLY//\//%2F}"; REPLY="${REPLY/#./%2E}"
	REPLY="${REPLY//</%3C}"; REPLY="${REPLY//>/%3E}"
	REPLY="${REPLY//\\/%5C}"
}
```


### run-markdown

```shell
# run-markdown file args...
# Compile `file` and source the result, passing along any positional arguments
run-markdown() {
	if [[ $BASH_VERSINFO == 3 ]]; then # bash 3 can't source from proc
		source /dev/fd/0 "${@:2}" <<<"$(mdsh-source "${1--}")"
	else source <(mdsh-source "${1--}") "${@:2}"
	fi
}
```

