---
layout: default
title: rpm.org - RPM Query Formats
---
# RPM Query Formats

As it is impossible to please everyone with one style of query output, RPM allows you to specify what information should be printed during a query operation and how it should be formatted.

## Tags

All of the information a package contains, apart from signatures and the actual files, is in a part of the package called the header. Each piece of information in the header has a tag associated with it which allows RPM to to tell the difference between the name and description of a package.

To get a list of all of the tags your version of RPM knows about, run the command `rpm --querytags`. It will print out a list like (but much longer then) this:

```
    RPMTAG_NAME
    RPMTAG_VERSION
    RPMTAG_RELEASE
    RPMTAG_EPOCH
    RPMTAG_SUMMARY
    RPMTAG_DESCRIPTION
    RPMTAG_BUILDTIME
    RPMTAG_BUILDHOST
    RPMTAG_INSTALLTIME
    RPMTAG_SIZE
```

As all of these tags begin with RPMTAG_, you may omit it from query format specifiers and it will be omitted from the rest of this documentation for the same reason.

A tag can consist of one element or an array of elements. Each element can be a string or number only.

### 64 Bit Tags

The number tags have traditionally been unsigned 32 bit integers only. With file and package sizes growing some 64 bit tags have been introduced. These should now be used instead of the old 32 bit ones without "LONG" at the beginning:

```
    LONGARCHIVESIZE - Uncompressed payload size
    LONGFILESIZES - Array of file sizes
    LONGSIGSIZE - Header + compressed payload size
    LONGSIZE - Sum of all file sizes 
```

If you have scripts using the old tags please change them. Packages that exceed the 4GB limit for any of those values will not contain the 32 bit tag. While the 64 bit tags are not physically present in "small" packages those tags are emulated for the queries.

## Query Formats

A query format is passed to RPM after the --queryformat argument, and normally should be enclosed in single quotes. This query format is then used to print the information section of a query. This means that when both -i and --queryformat are used in a command, the -i is essentially ignored. Additionally, using --queryformat implies -q, so you may omit the -q as well.

The query format is similar to a C style printf string, which the printf(2) man page provides a good introduction to. However, as RPM already knows the type of data that is being printed, you must omit the type specifier. In its place put the tag name you wish to print enclosed in curly braces ({}). For example, the following RPM command prints the names and sizes of all of the packages installed on a system:

```
    rpm -qa --queryformat "%{NAME} %{SIZE}\n"
```

If you want to use printf formatters, they go between the % and {. To change the above command to print the NAME in the first 30 bytes and right align the size to, use:

```
    rpm -qa --queryformat "%-30{NAME} %10{SIZE}\n"
```

## Arrays

RPM uses many parallel arrays internally. For example, file sizes and file names are kept as an array of numbers and an array of strings respectively, with the first element in the size array corresponding to the first element in the name array.

To iterate over a set of parallel arrays, enclose the format to be used to print each item in the array within square brackets ([]). For example, to print all of the files and their sizes in the slang-devel package followed by their sizes, with one file per line, use this command:

```
    rpm -q --queryformat "[%-50{FILENAMES} %10{FILESIZES}\n]" slang-devel
```

Note that since the trailing newline is inside of the square brackets, one newline is printed for each filename.

A popular query format to try to construct is one that prints the name of a package and the name of a file it contains on one line, repeated for every file in the package. This query can be very useful for passing information to any program that's line oriented (such as grep or awk). If you try the obvious,

```
    rpm --queryformat "[%{NAME} %{FILENAMES}\n]" cdp
```

If you try this, you'll see RPM complain about a "parallel array size mismatch". Internally, all items in RPM are actually arrays, so the NAME is a string array containing one element. When you tell RPM to iterate over the NAME and FILENAMES elements, RPM notices the two tags have different numbers of elements and complains.

To make this work properly, you need to tell RPM to always print the first item in the NAME element. You do this by placing a '=' before the tag name, like this:

```
    rpm --queryformat "[%{=NAME} %{FILENAMES}\n]" cdp
```

which will give you the expected output.

```
    cdp /usr/bin/cdp
    cdp /usr/bin/cdplay
    cdp /usr/man/man1/cdp.1
```

## Formatting Tags

One of the weaknesses with query formats is that it doesn't recognize that the INSTALLTIME tag (for example) should be printed as a date instead of as a number. To compensate, you can specify one of a few different formats to use when printing tags by placing a colon followed the formatting name after the tag name. Here are some examples:

```
    rpm -q --queryformat "%{NAME} %{INSTALLTIME:date}\n" fileutils
    rpm -q --queryformat "[%{FILEMODES:perms} %{FILENAMES}\n]" rpm
    rpm -q --queryformat \
    "[%{REQUIRENAME} %{REQUIREFLAGS:depflags} %{REQUIREVERSION}\n]" \
    vlock
```

The :shescape may be used on plain strings to get a string which can pass through a single level of shell and give the original string.

## Query Expressions

Simple conditionals may be evaluated through query expressions. Expressions are delimited by `%|...|`. The only type of expression currently supported is a C-like ternary conditional, which provides simple if/then/else conditions. For example, the following query format display "present" if the SOMETAG tag is present, and "missing" otherwise:

```
    %|SOMETAG?{present}:{missing}|
```

Notice that the subformats "present" and "missing" must be inside of curly braces.
Example: Viewing the Verify Flags

The following example query is run against dev because I know %verify is used there.

```
    rpm -q --qf '[%{filenames} %{fileverifyflags}\n]' dev
```

The flags are defined in rpmlib.h (check there for changes):

```
    #define RPMVERIFY_MD5           (1 << 0)
    #define RPMVERIFY_FILESIZE      (1 << 1)
    #define RPMVERIFY_LINKTO        (1 << 2)
    #define RPMVERIFY_USER          (1 << 3)
    #define RPMVERIFY_GROUP         (1 << 4)
    #define RPMVERIFY_MTIME         (1 << 5)
    #define RPMVERIFY_MODE          (1 << 6)
    #define RPMVERIFY_RDEV          (1 << 7)
```

A 1 bit in the output of the query means the check is enabled.
