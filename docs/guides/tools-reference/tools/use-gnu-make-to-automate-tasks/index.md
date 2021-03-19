---
slug: use-gnu-make-to-automate-tasks
author:
  name: Linode Community
  email: docs@linode.com
description: 'The make utility is not just for compiling C programs. Learn how to use it for automating other tasks.'
og_description: 'Two to three sentences describing your guide when shared on social media.'
keywords: ['list','of','keywords','and key phrases']
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 2021-03-10
modified_by:
  name: Linode
title: "Use GNU Make to Automate Tasks"
h1_title: "Use GNU Make to Automate Tasks"
contributor:
  name: Stephen Savitzky
  link: https://github.com/ssavitzky
external_resources:
- '[A web-focused Git
  workflow](http://joemaller.com/990/a-web-focused-git-workflow/) by Joe Maller'
- 'The [GNU make](https://www.gnu.org/software/make/manual/make.html) manual'
- '[Managing Projects with GNU Make, Third
  Edition](http://shop.oreilly.com/product/9780596006105.do) by Robert Mecklenburg'
---

[GNU make](https://www.gnu.org/software/make/manual/make.html) was originally
designed as a utility for building large programs -- it automatically
determines which parts need to be recompiled, and issues the commands needed
to recompile them. However, its uses have never been limited to compiling and
linking C programs -- it can also be a surprisingly effective scripting
language for automating any task that involves updating files or performing
actions whenever other files change.

{{< note >}}
In this guide `make`  or "Make" will always refer to GNU Make, the version
found on almost all Linux distributions.  It has a number of useful
extensions -- pattern rules and simply-expanded variables. Sections
[14](https://www.gnu.org/software/make/manual/make.html#Features) and
[15](https://www.gnu.org/software/make/manual/make.html#Missing) of the [GNU
make Manual](https://www.gnu.org/software/make/manual/make.html) describe GNU
Make's differences from the BSD and System V versions. 
{{< /note >}}

FIXME add installation?
QUERY: start with a more complex makefile and identify the parts?

## Make Basics

Make starts by reading a file called `makefile`  that tells it what to do.  The
makefile contains "rules" that describe files to be made -- "targets" -- the
prerequisites that may have to be built first, and the "recipe" for rebuilding
the targets.  So a typical rule looks like:

```
target... : prerequisite...
	recipe...
```

Targets and prerequisites are separated by spaces; spaces around the colon are
ignored.  The recipe is a series of shell commands, each preceeded by a TAB
character. This needs to be a literal tab, not some number of spaces; any text
editor that understands the format of makefiles will warn you by showing
spaces in the recipe in a different color.

Each line of the recipe is run in a separate shell process, after removing the
leading tab; shell commands can be separated by `;` (semicolon), and long
lines can be continued by following each one but last with a `\` (backslash).
The recipe is run whenever the target does not exist, or if any of its
prerequisites has been modified since the last time it was built.


{{< note >}}
For portability, all versions of make use `/bin/sh` as the default shell. You
can change it by assigning to the `SHELL` variable -- `/bin/bash` is another
popular choice. Make inherits almost all of the environment variables exported by
the shell that invokes it; `SHELL` is an exception.
{{< /note >}}


### Rules and Prerequisites

As an oversimplified example, this is a makefile for an extremely simple
website:

{{< file "makefile" >}}
# everything between a hash-mark and the end of the line is ignored
site/index.html: index.md
	pandoc -s -o site/index.html index.md
{{< /file >}}

This makefile has one rule; make uses the first rule it encounters in the file
as the default.  Now every time `index.md` is modified, make will determine
that `site/index.html` is older, and re-make it.

There are two problems with this. The first is that the first time we run
Make in this directory it will fail because there is no directory called
`site`. We can fix this by adding an _order-only prerequisite_, separated from
the ordinary prerequisites by a `|`. Make will build order-only prerequisites
if they don't exist, but will not try to re-build them if they get out of
date.  They can also be useful for making sure required applications are
installed. 

{{< file makefile >}}
site/index.html: index.md | site
	pandoc -s -o site/index.html index.md

site:
	mkdir -p site
{{< /file >}}

Note that a rule with no prerequisites will only be run if the target does not
exist.  Order-only prerequisites won't be re-built even if they have
prerequisites of their own.

The second problem is that, if we want to add an "about" page, the rule to
make it won't be the first in the makefile, so make won't try to run it unless
we mention _both_ targets on the command line.  One way to solve this is to
add another rule that depends on both pages:

{{< file makefile >}}
.PHONY: build
build:: site/index.html site/about.html

site/index.html: index.md | site
	pandoc -s -o site/index.html index.md

site/about.html: about.md | site
	pandoc -s -o site/about.html about.md

site:
	mkdir -p site
	
build::
	echo build complete
{{< /file >}}

The `.PHONY` target tells make that `build` isn't a real file, just the name
of a recipe.  Even if a file called `build` gets added to the directory, make
will still run the recipe when you give `build` as a goal on the make command
line or as a prerequisite of some other target.

We have used a _double-colon_ rule for build; notice that there are two rules
with `build` as their target. Each double-colon rule has its prerequisites
checked independently -- that makes it easy to add additional rules for the
same target but different prerequisites. Unlike a single-colon rule, the
recipe for a double-colon rule with no prerequisites will _always_ be run,
even if the file exists.

For example,

```
built-on.md::
	date > $@
```

This uses an _automatic variable_, `$@`, which evaluates to the name of the
target. We'll see more automatic variables in the next section.


### Implicit Rules

We can start simplifying and, more importantly, generalizing this by replacing
the two _explicit_ rules by a single _implicit_ rule.  Make has many built-in
implicit rules, for programs and documents written in several different
languages. You can say `make foo` in a directory containing a file called
`foo.c` and make will be able to deduce that it needs to use the C compiler to
make it.  Make doesn't include a built-in rule for turning markdown into HTML,
but you can write a _pattern rule_ that does that.

A pattern rule has a target that contains a single `%` character, which
matches a non-empty substring of the target. That substring, called the
_stem_, is substituted for `%` wherever it appears in the prerequisites. So we
can make the HTML files with a rule like:

```
site/%.html: %.md | site/
	pandoc -s -o $@ $<
```

In addition to `$@`, which evaluates to the target, this rule uses the
automatic variables `$<`, which evaluates to the first prerequisite of the
rule in which it appears. You'll find the complete list of automatic variables
in [Section 10.5.3 of the
manual](https://www.gnu.org/software/make/manual/make.html#Automatic-Variables).

{{< note >}}
Because `$` is used for variable expansion in both Make and the shell, you
pass a dollar sign to the shell in a recipe by doubling it.  (There's nothing
special about this: Make has a built-in variable `$` that evaluates to the
string "`$`".)
{{< /note >}}

### Variables

Another way to simplify and generalize a `makefile` is with _variables_. One
obvious use for a variable is the list of targets:

{{< file makefile >}}
pages = site/index.html site/about.html
build: $(pages)
site/%.html: %.md | site
		pandoc -s -o $@ $<
{{< /file >}}

{{< note >}}
If the parentheses around the variable name are missing, the variable name
will be the single character after the dollar sign (which is why most of the
automatic variables used in rules, like `$@`, have single-character names).
{{< /note >}}

Make also uses variables (actually more like constants, so it's a common
practice to use uppercase for these) in place of program
names, so we might use `PANDOC` to point to `/usr/bin/pandoc`. It might be
better in this case to use something generic like `MARKDOWN` in case we want
to change it; it makes sense to include the `pandoc`-specific option `-s`
(`--standalone`) in the variable's definition, since other markdown processors
are likely to use a different set of options.

{{< file makefile >}}
MARKDOWN = /usr/bin/pandoc -s
pages = site/index.html site/about.html
build: $(pages)
site/%.html: %.md | site
		$(MARKDOWN) -o $@ $<
{{< /file >}}

The variable `MAKE` is treated specially -- Make uses it to identify
recursive statements in recipes that it needs to execute even if the `-n`
(`--just-print`) option was set on the command line, telling Make to "just
print" recipes without executing them.  You can find more about using Make
recursively in [Section 5.7 of the
manual](https://www.gnu.org/software/make/manual/make.html#Recursion)

### Functions

Listing all the targets in a project makes sense for something like a C
program, but it's tedious and error-prone for a website.  Fortunately, Make
has _functions_ for manipulating filenames and other text. Start with
`$(wildcard *.md)`, which evaluates to a list of all files in the current
directory with a `.md` extension. Then use the `subst` function to replace
`.md` with `.html`, and `addprefix` to add the destination directory. Putting
it all together, we have:

```
pages = $(addprefix site/, $(subst .md,.html,$(wildcard *.md)))
```

Note that instead of `subst` we could have used `basename` and `addsuffix`:

```
pages = $(addprefix site/, $(addsuffix .html, $(basename $(wildcard *.md))))
```

The `shell` function lets you use shell "one-liners" to do things that are
difficult or impossible with Make's somewhat limited set of functions. (It's
better to do things in Make if you can, because it's faster and often
significantly more readable.) For example,

```
DATE := $(shell date --rfc-3339=seconds)
```

You can find the complete set of functions in [Section 8 of the
manual](https://www.gnu.org/software/make/manual/make.html#Functions).

This also demonstrates GNU Make's second kind of variable assignment.
Variables assigned with `=` are called "recursively expanded variables"; their
value is parsed, and any variables or functions they contain are expanded,
every time they are used. Most of the timethat's what you want; an expression
like `$(basename $@)` would be useless unless it was expanded in every recipe
it's used in.

"Simply expanded variables" are assigned using `:=` or `::=` (the second is
the only form recognized by most other versions of Make); any variables or
functions they contain are expanded only once, at the point where the varable
is assigned. Using a simply-expanded variable for `DATE` ensures that, no
matter how many times we use it, it will always expand to the same string for
a given run of Make.


## Updating a website with Make and Git

In '[A web-focused Git
workflow](http://joemaller.com/990/a-web-focused-git-workflow/)', Joe Maller
describes a popular way of updating a web site that uses a bare repository on
the same server as the site, with a `post-update` hook that pulls from the
bare repo into the site's root directory whenever a developer pushes an
update. Simply pulling will work as long as all of the website's files are
under git control, but in most cases they will be build products. That's where
Make comes in:

### A versatile post-update hook

If Git finds an executable file called `/hooks/post-update` in its
repository, it will be run whenever that website is pushed to. The command
line will contain the names of all the refs pushed.  This post-update hook
runs `make` in the website's directory after changes are pushed.

{{< file "website.git/hooks/post-update" bash >}}
#!/bin/bash
# this hook is written assuming that `website.git` and `website` are 
# siblings, i.e. both are subdirectories of the same parent.
unset GIT_DIR; export GIT_DIR
for ref in $*; do
	# The refs being updated are passed on the command line.
	# We are only interested in main.
    if [ "$ref" = refs/heads/main ]; then
		# --ff-only ensures that un-pushed changes won't be overwritten
		git -C ../website pull --ff-only
		# If the website has a makefile, make build
		[[ -f ../website/makefile ]] && make -C ../website build
	fi
done
{{< /file >}}

The `-C` option in both Git and Make tells them to change to the indicated
directory. Building on the server means that derived files, some of which can
be large, don't have to be under git control.

### Makefiles for static site builders

Static site builders like Jekyll (which is used for GitHub Pages), Hugo, and
so on, do a great deal for you. But if you're supporting more than one site,
and they're built with different builders, you can use Make to give them all
a uniform command-line interface. Here is a simple set of recipes for a Jekyll
site that can be deployed by pushing either to GitHub Pages, or to a staging
server that uses the hook shown in the previous section. The corresponding
remotes are called `github` and `staging`.

```
serve:                       # run a local server for initial testing
	jekyll serve

staging:
	git status
	git push staging main

# this target is invoked by the post-update hook on the staging server
build:
	jekyll JEKYLL_ENV=production build

prod:
	git checkout production
	git merge --ff-only --no-edit main
	git tag -a -m "pushed to production $$(date)"
	git push github production
	git checkout main
```

The recipe for `prod` automates all of the git housekeeping around making a
release.  GitHub builds Jekyll pages and serves them automatically. The
`staging` recipe uses the hook in the previous section to build a copy of the
site on your Linode for further testing.

Of course you could go farther and use Make as your static site builder -- the
first section of this guide might give you an idea of how to start.

## Conclusion

This guide covers only a few of GNU Make's potential uses (and only a few
of its many features). Make's combination of declarative rules, filename- and
string-processing functions, and shell scripting make it a versatile addition
to any programmer's or system administrator's toolkit.
