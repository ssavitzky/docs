---
slug: git-subtree
author:
  name: Linode Community
  email: docs@linode.com
description: 'This guide discusses the git-subtree subcomand and its uses for merging and splitting repositories, managing dependencies, and refactoring.'
keywords: ['git subtree', 'git submodule alternative', 'refactoring']
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 
modified_by:
  name: Linode
title: "How to Use Git Subtree"
h1_title: "How to Use Git Subtree"
enable_h1: true
contributor:
  name: Stephen Savitzky
  link: https://github.com/ssavitzky/
external_resources:
- [Ubuntu Manpage: git-subtree - Merge subtrees together and split repository into subtrees](http://manpages.ubuntu.com/manpages/trusty/en/man1/git-subtree.1.html)
- [Git Subtree | W3Docs Online Tutorial](https://www.w3docs.com/learn-git/git-subtree.html)
- [Git Glossary](https://git-scm.com/docs/gitglossary)
---

== Introduction

{{< note >}}
This guide uses "tree" and "subtree" to refer to the data structures in your
Git repository that correspond to directories and subdirectories in your
filesystem.  Like most Git documentation it will occasionally use "tree"
etc. to refer to directories (e.g., "working tree"). See the definitions of
[tree object](https://git-scm.com/docs/gitglossary#def_tree_object)
and[working tree](https://git-scm.com/docs/gitglossary#def_working_tree)
in the [Git Glossary](https://git-scm.com/docs/gitglossary). It also follows
the [git-subtree man
page](http://manpages.ubuntu.com/manpages/trusty/en/man1/git-subtree.1.html)
in using "command" to refer to the sub-subcommands `add`, `merge`, `pull`, and
`split`.
{{< /note >}}

Subtrees let you include a subproject -- a library, for example -- as a
subdirectory of your main project, optionally including the subproject's
entire history. After adding a subtree, you can push and pull changes to and
from its upstream repository. Unlike submodules, which have a similar
function, the new subtree simply becomes a part of your project.  It requires
no metadata (e.g. `.gitmodules` files) in your repository, and no changes to
your workflow. You can share your repository without requiring your users to
do anything special or understand how subtrees work.

You can also use `git subtree` to make a subdirectory, along with all of its
history, into a stand-alone project.

=== Synopsis:

```
       git subtree add   -P <prefix> [OPTIONS] <commit>
       git subtree add   -P <prefix> [OPTIONS] <repository> <ref>
       git subtree pull  -P <prefix> [OPTIONS] <repository> <ref>
       git subtree push  -P <prefix> [OPTIONS] <repository> <ref>
       git subtree merge -P <prefix> [OPTIONS] <commit>
       git subtree split -P <prefix> [OPTIONS] [<commit>]
```

The `-P <prefix>` (or `--prefix=<prefix>`) argument is mandatory; it specifies
the path from the root of the working tree to the subtree to be operated on.
The leading and trailing slashes are omitted, and if the path contains
multiple directory components they must be separated by forward slashes even
if you are working on Windows.

For the add, pull, and push commands it's usually convenient to create a
remote that points to the repository.

== Split -- turn a subtree into a tree

The `split` command traverses your Git history and identifies all of the
commits that refer to a specified subtree. It then constructs a synthetic
history with the contents of the subtree as its root, which can thus be
exported as an independent Git repository.  There are other ways to do this,
e.g. `git-filter-repo` or `git-filter-branch`, but `git subtree split` is by
far the simplest if you can deal with one subtree at a time. Among other
things, it's non-destructive, so you don't have to clone your repo before you
start, the way you do with the filter commands.

After splitting, the command prints a single commit id to standard output,
corresponding to the HEAD of the new tree, and optionally creates a branch
there (with the `-b` or `--branch` parameter).  Repeated splits of the same
history are guaranteed to be identical (have the same commit ids), so that if
anyone makes changes that affect the subtree, another split will be an
extension of the previous one. That means that you don't have to worry about
other people working on the same subtree.

With the `--rejoin` option, `split` creates a merge commit that joins the
just-extracted history with your main one. This makes it easier to split again
after you've made edits, but has the disadvantage that every commit affecting
the subtree will appear twice in your log.  You can use the
`--annotate=<annotation>` option to add `<annotation>` as a prefix to each of
the new commit messages and use `grep -v` to hide them.

After splitting, the original subtree is still part of your project -- all
you've done is make a clean copy of it in a separate branch.

=== Using split

By far the most common use of `git subtree split` is extracting a subtree from
a large project and putting it in a new repository so that it can be shared
with other developers and maintained independently. This is often done with a
library that you think other developers will want to use, or that _you_ want
to use, or with a utility that's part of a larger project that you want to use
for some other purpose.  You can start by making an empty, bare repo for the
subtree:

```
$ mkdir new-library
$ cd new-library
$ git init --bare
```

Then go back to the large project and split off the part you want:

```
$ cd big-project
$ git subtree split --prefix=library -b split
```

Now you can push the subtree into the new repository:

```
$ git push path/to/new-library split:main
```

Now, if you're ready, you can delete the subtree from the main project using
`git rm -r`.


=== Keeping your subtree's history clean

The `split` command creates a new history that includes only commits that
refer to the subtree being split. It does _not_ rewrite your commit
_messages_. If you make a change that affects both the subtree and something
else in your project, it may be a good idea to break that into two pieces, so
that the commit messages will still make sense after being split.

It isn't necessary, though, especially if the commit message would make sense
without being split into parts, for example if you're applying a formatting
change to everything in your repository.


== Add -- turn a tree into a subtree

```
git subtree add -P <prefix> <commit>
git subtree add -P <prefix> <repository> <ref>
```

The `add` command creates the `<prefix>` subtree by importing its contents and
history from a separate tree (typically a branch in some repository).  It then
creates a commit that joins the imported subtree's history with your own. With
the `--squash` option, it imports only a single commit instead of the whole
history -- that's usually what you need.

In both cases the new commit contains enough information so that if you create
new commits in the subtree, a later `split` will be able to construct a
history that will merge cleanly onto the tree you imported.

=== Using add

Now suppose some other developer wants to use your library in another
project.  They do that with:

```
$ git remote add newlib git@github.com:yourname/new-library
$ git subtree add --prefix=new-library --squash newlib main
```

(It's almost always convenient to create a remote that points to the library's
repository; that makes subsequent pushes and pull much simpler.)

The `--squash` option brings in the files from the library and creates a
single commit, without also adding their history.  That's almost always what
you want.  In addition, because it's a single commit, you can get add any
version of the subproject, and go forward and backward in history.

== Merge, Pull, and Push

```
git subtree pull  -P <prefix> <repository> <ref>
git subtree push  -P <prefix> <repository> <ref>
git subtree merge -P <prefix> <commit>
```

The `merge` command merges recent changes up to the given commit and merges
them into the `<prefix>` subtree. Just like `git merge`, this doesn't remove
your own local changes. With the `--squash` option, it creates a single commit
that contains all of the changes. In that case, the merge direction doesn't
have to be forward; you can merge any commit.

In most cases you'll want to use `pull` to update a subtree; this parallels
`git pull` and fetches the given ref from the given remote repository. Also
like `merge`, in most cases you will be using `--squash` to pull only the
files attached to the given commit, rather than 

The converse of `pull` is `push`, which does a `split` and pushes the result
to the remote repository. This is particularly useful if you are in the
process of splitting off a library, but one of your coworkers makes their own
changes to the subtree in the larger project.  You can pull their changes
(which may include changes outside the subtree), merge them, and push the
resulting subtree to the split-off library's repo.

== When to use something else

Git subtree is most useful in two situations:

  * Splitting a subtree off of a larger project and making it a stand-alone
	project.
  * Including a separate project as a subtree so that you can share a project
	without requiring the people you're sharing it with to know anything about
	subtrees or make any changes in their own workflow.

If you want to make a more complex stand-alone project, containing multiple
subtrees (a library plus tests or documentation) or a subtree plus additional
files from the top level (for example a complex Makefile or build scripts),
you will probably be better off making a clone and stripping out the parts you
don't need using `git filter-repo`. This is especially true if you're
splitting up a large source tree into multiple small ones, with no plan to keep
the combined project together. That's a common problem when the large tree
came from a `svn` repo shared across a large organization.

Another option is to create a patch file using `git log`:

```
# create a patch file:  replace PATHS with the list of paths you want to copy
$ git log --pretty=email --patch-with-stat --reverse --full-index --binary \
          -- PATHS > /tmp/patches
# now go to the place where you want the files to end up:
$ cd ../destination
# ... and apply the patches.
$ git am /tmp/patches
```

You can also use `git format-patch`; it creates a separate patch file for each
commit, which can be a problem if there are many of them, but which would make
it easier to make changes to specific patches. The other advantage of making a
single patch file is that you can use `sed` or a text editor to modify the
pathnames.  That would look something like:

```
$ sed -i -e 's@deep/path/that/you/want/shorter@short/path@g' /tmp/patches
```

By using the empty string as the short path, you will end up doing essentially
the same thing as `split`, putting the contents of the folder at the top level
of the new repository, but for something that simple you may as well use `git
subtree`. If you want to do something more complicated, like move multiple
subtrees and keep their commits in the same order, or add a subtree to another
project without complicating its history, use a patch file.
