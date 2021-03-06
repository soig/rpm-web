---
layout: default
title: rpm.org - Macro syntax
---
# Macro syntax

RPM has fully recursive spec file macros. Simple macros do straight text substitution. Parameterized macros include an options field, and perform argc/argv processing on white space separated tokens to the next newline. During macro expansion, both flags and arguments are available as macros which are deleted at the end of macro expansion. Macros can be used (almost) anywhere in a spec file, and, in particular, in "included file lists" (i.e. those read in using %files -f \<file\>). In addition, macros can be nested, hiding the previous definition for the duration of the expansion of the macro which contains nested macros.

## Defining a Macro
To define a macro use:

```
    %define <name>[(opts)] <body>
```

All whitespace surrounding <body> is removed. Name may be composed of alphanumeric characters, and the character `_' and must be at least 3 characters in length. A macro without an (opts) field is "simple" in that only recursive macro expansion is performed. A parameterized macro contains an (opts) field. The opts (i.e. string between parentheses) is passed exactly as is to getopt(3) for argc/argv processing at the beginning of a macro invocation. "--" can be used to separate options from arguments. While a parameterized macro is being expanded, the following shell-like macros are available:

```
    %0      the name of the macro being invoked
    %*      all arguments (unlike shell, not including any processed flags)
    %**     all arguments (including processed flags)
    %#      the number of arguments
    %{-f}   if present at invocation, the flag f itself
    %{-f*}  if present at invocation, the argument to flag f
    %1, %2  the arguments themselves (after getopt(3) processing)
```

At the end of invocation of a parameterized macro, the above macros are (at the moment, silently) discarded.

## Writing a Macro
Within the body of a macro, there are several constructs that permit testing for the presence of optional parameters. The simplest construct is "%{-f}" which expands (literally) to "-f" if -f was mentioned when the macro was invoked. There are also provisions for including text if flag was present using "%{-f:X}". This macro expands to (the expansion of) X if the flag was present. The negative form, "%{-f:Y}", expanding to (the expansion of) Y if -f was *not* present, is also supported.

In addition to the "%{...}" form, shell expansion can be performed using "%(shell command)". The expansion of "%(...)" is the output of (the expansion of) ... fed to /bin/sh. For example, "%(date +%%y%%m%%d)" expands to the string "YYMMDD" (final newline is deleted). Note the 2nd % needed to escape the arguments to /bin/date.

## Builtin Macros
There are several builtin macros (with reserved names) that are needed to perform useful operations. The current list is

```
  %trace              toggle print of debugging information before/after
                      expansion
  %dump               print the active (i.e. non-covered) macro table
  %verbose            is rpm in verbose mode?

  %{echo:...}         print ... to stdout
  %{warn:...}         print ... to stderr
  %{error:...}        print ... to stderr and raise an error
 
  %define ...         define a macro
  %undefine ...       undefine a macro
  %global ...         define a macro whose body is available in global context

  %{expand:...}       like eval, expand ... to <body> and (re-)expand <body>
  %{shrink:...}       trim leading and trailing whitespace, reduce
                      intermediate whitespace to a single space (in >= 4.14.0)
  %{quote:...}        quote a parametric macro argument, needed to pass
                      empty strings or strings with whitespace (in >= 4.14.0)

  %{lua:...}          expand with the [embedded Lua interpreter](lua.html)

  %{uncompress:...}   expand ... to <file> and test to see if <file> is
                      compressed.  The expansion is
                          cat <file>                # if not compressed
                          gzip -dc <file>           # if gzip'ed
                          bzip2 -dc <file>          # if bzip'ed
  %{basename:...}     basename(1) macro analogue
  %{dirname:...}      dirname(1) macro analogue
  %{suffix:...}       expand to suffix part of a file name
  %{url2path:...}     convert url to a local path
  %{getenv:...}       getenv(3) macro analogue
  %{getconfdir:...}   expand to rpm "home" directory (typically /usr/lib/rpm)

  %{S:...}            expand ... to <source> file name
  %{P:...}            expand ... to <patch> file name
  %{F:...}            expand ... to <file> file name
```

Note that %define and %global differ in more ways than just scope: the body of a %define'd macro is lazily expanded (ie when used), but the body of %global is expanded at definition time. It's possible to use %%-escaping to force lazy expansion of %global.

Macros may also be automatically included from /usr/lib/rpm/macros. In addition, rpm itself defines numerous macros. To display the current set, add "%dump" to the beginning of any spec file, process with rpm, and examine the output from stderr.

## Example of a Macro
Here is an example %patch definition from /usr/lib/rpm/macros:

```
    %patch(b:p:P:REz:) \
    %define patch_file  %{P:%{-P:%{-P*}}%{!-P:%%PATCH0}} \
    %define patch_suffix    %{!-z:%{-b:--suffix %{-b*}}}%{!-b:%{-z:--suffix %{-z*}}}%{!-z:%{!-b: }}%{-z:%{-b:%{error:Can't specify both -z(%{-z*}) and -b(%{-b*})}}} \
        %{uncompress:%patch_file} | patch %{-p:-p%{-p*}} %patch_suffix %{-R} %{-E} \
    ...
```

The first line defines %patch with its options. The body of %patch is

```
    %{uncompress:%patch_file} | patch %{-p:-p%{-p*}} %patch_suffix %{-R} %{-E}
```

The body contains 7 macros, which expand as follows

```
    %{uncompress:...}       copy uncompressed patch to stdout
      %patch_file           ... the name of the patch file
    %{-p:...}               if "-p N" was present, (re-)generate "-pN" flag
      -p%{-p*}              ... note patch-2.1 insists on contiguous "-pN"
    %patch_suffix           override (default) ".orig" suffix if desired
    %{-R}                   supply -R (reversed) flag if desired
    %{-E}                   supply -E (delete empty?) flag if desired
```

There are two "private" helper macros:

```
    %patch_file     the gory details of generating the patch file name
    %patch_suffix   the gory details of overriding the (default) ".orig"
```

## Using a Macro
To use a macro, write:

```
    %<name> ...
```

or

```
    %{<name>}
```

The %{...} form allows you to place the expansion adjacent to other text. The %\<name\> form, if a parameterized macro, will do argc/argv processing of the rest of the line as described above. Normally you will likely want to invoke a parameterized macro by using the %\<name\> form so that parameters are expanded properly.

Example:

```
    %define mymacro() (echo -n "My arg is %1" ; sleep %1 ; echo done.)
```

Usage:

```
    %mymacro 5
```

This expands to:

```
    (echo -n "My arg is 5" ; sleep 5 ; echo done.)
```

This will cause all occurrences of %1 in the macro definition to be replaced by the first argument to the macro, but only if the macro is invoked as "%mymacro 5". Invoking as "%{mymacro} 5" will not work as desired in this case.

## Command Line Options
When the command line option "--define 'macroname value'" allows the user to specify the value that a macro should have during the build. Note lack of leading % for the macro name. We will try to support users who accidentally type the leading % but this should not be relied upon.

Evaluating a macro can be difficult outside of an rpm execution context. If you wish to see the expanded value of a macro, you may use the option

```
    --eval '<macro expression>'
```

that will read rpm config files and print the macro expansion on stdout.

Note: This works only macros defined in rpm configuration files, not for macros defined in specfiles. You can use %{echo: %{your_macro_here}} if you wish to see the expansion of a macro defined in a spec file.

## Configuration using Macros
Most rpm configuration is done via macros. There are numerous places from
which macros are read, in recent rpm versions the macro path can be seen
with `rpm --showrc|grep "^Macro path"`. If there are multiple definitions
of the same macro, the last one wins. User-level configuration goes
to ~/.rpmmacros which is always the last one in the path.

The macro file syntax is simply:

```
%<name>		<body>
```

...where <name> is a legal macro name and <body> is the body of the macro.
Multiline macros can be defined by shell-like line continuation, ie `\`
at end of line.

Note that the macro file syntax is strictly declarative, no conditionals
are supported (except of course in the macro body).

## Macro Analogues of Autoconf Variables
Several macro definitions provided by the default rpm macro set have uses in packaging similar to the autoconf variables that are used in building packages:

```
    %_prefix            /usr
    %_exec_prefix       %{_prefix}
    %_bindir            %{_exec_prefix}/bin
    %_sbindir           %{_exec_prefix}/sbin
    %_libexecdir        %{_exec_prefix}/libexec
    %_datadir           %{_prefix}/share
    %_sysconfdir        %{_prefix}/etc
    %_sharedstatedir    %{_prefix}/com
    %_localstatedir     %{_prefix}/var
    %_libdir            %{_exec_prefix}/lib
    %_includedir        %{_prefix}/include
    %_oldincludedir     /usr/include
    %_infodir           %{_prefix}/info
    %_mandir            %{_prefix}/man
```

