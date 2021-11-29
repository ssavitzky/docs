---
slug: how-to-define-new-git-subcommands
author:
  name: Linode Community
  email: docs@linode.com
description: 'You can easily define new Git subcommands with aliases and scripts'
og_description: 'You can easily define new Git subcommands with aliases and scripts'
keywords: ['list','of','keywords','and key phrases']
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 
modified_by:
  name: Linode
title: "How to Define New Git Subcommands"
h1_title: "How to Define New Git Subcommands"
enable_h1: true
contributor:
  name: Stephen Savitzky
  link: https://github.com/ssavitzky/
external_resources:
- [Git - git Documentation](https://git-scm.com/docs/git)
- [Git - git-config Documentation](https://git-scm.com/docs/git-config)
- [Git - gitcore-tutorial Documentation](https://git-scm.com/docs/gitcore-tutorial)
- [tj/git-extras: GIT utilities](https://github.com/tj/git-extras)
- [Must Have Git Aliases: Advanced Examples](https://www.durdn.com/blog/2012/11/22/must-have-git-aliases-advanced-examples/)
- [Pro Git section 2.7 - Git Aliases](https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases) 
---

== Introduction

Git has a lot of subcommands (which you can list with `git --list-cmds`).
Some of them have odd differences in their syntax -- why should you use `git
branch -d` to delete a branch, but `git remote rm` to delete a remote? Why do
you use the `-c` option to create a branch with `git-switch`, when you're used
to using the `-b` option with `git-checkout`? Part of the reason is that, like
Unix, Git is ridiculously easy to customize.

There are two ways to customize Git: aliases, which Git finds in its config
files, and programs, which Git finds by looking along your `$PATH` for
executables programs with names that start with `git-`. This is almost the
same as the way shells like Bash can be extended; the difference is that Git
has multiple config files, which lets you make repo-specific aliases.

== Aliases

Aliases let you define short names for longer Git commands. The easiest way to
define them is to edit your `.gitconfig` file, which located in your home
directory on Unix and Linux systems, and add the definition in the `[alias]`
section. For example,

```
[alias]
    st = status
    amend = commit -a --amend
```

defines `st` as an alias for `status`, and `amend` as an alias for `commit -a
--amend` -- the alias is simply replaced by its definition following the `git`
command. Anything after the alias on the command line comes after the
definition, so the following two commands are equivalent:

```
$ git amend --no-edit
$ git commit -a --amend --no-edit
```

The definition of an alias doesn't have to start with a subcommand; it can
include parameters that come before the subcommand as well as after it. So you
can use a definition like

```
    ps = --paginate status
```

to define `ps` as a version of `git status` that paginates its output by
running it through `less`, which is not its default.  You can find the whole
list of of options that can precede the subcommand in Git's `man` page.  One
of the more useful ones is `-c <name>=<value>` to set a configuration
variable.


=== Aliases for shell commands

Git aliases aren't confined to Git subcommands and their options. An alias
prefixed with an exclamation point is passed directly to the shell instead of
to Git. For example,

```
    k = !gitk --all&
    top = !pwd
```

The `k` alias runs the GUI repository browser `gitk` in the background.  The
`top` alias prints out the top level of the working tree, because that's where
Git runs shell aliases.  As with ordinary aliases, the expanded alias is followed by
whatever arguments followed the alias on the command line. For example,

```
    f = !git ls-files | grep
```

finds filenames that contain a given string. (This example comes from "[Must
Have Git Aliases: Advanced
Examples](https://www.durdn.com/blog/2012/11/22/must-have-git-aliases-advanced-examples/)"
by Nicola Paolucci, which contains many other useful aliases.)

```
    run = "!pwd;"
```

Prints out the top-level directory, then if there's anything else in the
command-line it will follow the semicolon and so be taken as a command to
run. The quotes are needed because an unquoted semicolon makes the rest of the
line into a comment.  (Hash also starts a comment, just as it does in most scripting
languages. Using semicolon this way comes from Lisp and some assembly
languages.)

Sometimes you need the command-line arguments someplace other than the end of
the command.  Although you could do that by defining a subcommand with a
stand-alone script (see the last section), you can often handle simple cases
by defining and invoking a shell function.  So in the previous example, you
might want to pass an argument to `git-ls-files`; for example `--modified`.
You can do that with

```
    g = "!f () { git ls-files $2 | grep $1; }; f"
```

(Note that if you want to pass options to `grep` you can do so by quoting the
first argument, e.g. `git g "-i foo"`.)

=== Local aliases

You can, of course, use the `git-config` subcommand to define aliases from the
command line without using an editor.  For example,

```
    $ git config --global alias.ls 'log --oneline'
```

Without the `--global` parameter, `git-config` will make its configuration
changes in the local repository, i.e. in `.git/config`.  This can be useful if
you want to define an easy-to-remember alias that does different things in
different projects. For example,

```
    $ git config build '!make'  # in a C project
    $ git config build '!ant'   # in a Java project
    $ git config build '!rake'  # in a Ruby project
```

It's convenient that git runs shell aliases in the top level of the work
tree.  Local aliases are also convenient when experimenting with Git aliases!
When you're finished, clean up with, e.g.,

```
    $ git config --unset alias.build
```


== Scripts (and other programs)

Aliases work for many simple extensions, but more complicated subcommands are
best implemented as separate programs, often written in Bash or some other
scripting language.

Adding a new subcommand to Git is almost exactly the same as adding a new
command to your operating system: write a program that does what you want,
make it executable, and put it in a directory mentioned in `$PATH` --
typically `~/bin` or `/usr/local/bin`. Git only does two things differently
from what Bash or any other shell does: it gets the program name by prepending
`git-` to the subcommand name, and it looks in some high-priority places first
before looking in the directories listed in `$PATH` (which keeps you from
inadvertently hiding one of Git's core commands).

You can find out where Git keeps its core commands with the `--exec-path`
option.  On most Linux systems that will be `/usr/lib/git-core`:

```
$ git --exec-path 
/usr/lib/git-core
```

=== Example

As a rather trivial example, you can re-implement the first example under
aliases with a shell script, and improve it a little by passing `gitk` the
options on the rest of the command line, making `--all` the default if no
arguments are given.

``` bash
#!/bin/bash
#  run gitk in the background.
#  Defaults to --all if no parameters provided on the command line

if [ -z "$*" ]; then
    gitk --all &
else
    gitk $* &
fi
```

=== More useful examples

There is a good collection of Git subcommands called `git-extras`.  Not only
are they useful, they make a great set of examples. Most Linux distributions
have it packaged under that name, but the easiest way to look at the code for
them is on GitHub, at <a href="https://github.com/tj/git-extras"
>https://github.com/tj/git-extras</a>.

Several years ago there was a meme going around that said

```
In case of fire:
  git commit
  git push
  leave the building
```

Needless to say, that was quickly implemented as a subcommand, which you can
find at <a href="https://github.com/qw3rtman/git-fire" >qw3rtman/git-fire:
Save Your Code in an Emergency</a>.

=== Original scripted subcommands

The [Gitcore Tutorial](https://git-scm.com/docs/gitcore-tutorial) has a wealth
of information for developers working in and around the Git codebase, which of
course includes developing subcommands.  It mentions that many of Git's
subcommands were originally implemented as scripts.  Since then most have 
been re-written in C and turned into built-ins, but until recently, the Git
repository included many of these original subcommand scripts in
`contrib/examples`. The simplest way to see them is to look at
[git/contrib/examples at 90bbd50](https://github.com/git/git/tree/90bbd502d54fe920356fa9278055dc9c9bfe9a56/contrib/examples),
which is the parent of [the commit that removed
them](https://github.com/git/git/commit/49eb8d39c78f161231e63293df60f343d208f409).
Since then the original scripts have become increasingly obsolete, but they
are still good examples of how to write subcommands.

