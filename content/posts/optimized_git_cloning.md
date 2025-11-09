+++
title = "Optimized git-Repository Retrieval"
description = "Often, git repositories are small enough to just retrieve a full copy of the upstream repository. If that becomes too painfull, git still got you covered."
date = 2025-11-08T23:00:00+01:00
type = "post"
tags = ["git", "clone", "repository", "speedup", "optimization"]
draft = false
showTableOfContents = true
+++

If a `git` repository is a few hundred mega bytes or more in size, cloning and fetching becomes time and resource consuming.
Especially CI environments perform regular clones of a repository and benefit from optimizations.

`git` provides advanced settings to download only the necessary files and save time and space in the process.
This post demonstrates various optimizations and measures the speedups with the example of [LLVM](https://github.com/llvm/llvm-project).
The baseline of cloning `llvm` is several minutes, downloading `6GB` of compressed data.

## TL;DR: `scalar` For Developers

The `git` package includes the [`git-scalar`](https://git-scm.com/docs/scalar) subcommand that is available as distinct exectuable `scalar` too.
This command performs various setting adjustments useful for big repositories.
Not all tricks described in this post are directly accessible, e.g. history reduction, and various optimization make only sense in long lived repositories clones.
It is still the easiest and fastest way to get a faster `git` repository for your development on a big (mono) repo.
Automation jobs sometimes require more control and `scalar clone` may not be the best match.

## Reducing The History

The most common reduction is to download only parts of the history, called a _shallow clone_.
Reducing the history reduces the amount of data to be downloaded drastically.
```bash
$ time git clone --depth=1 git@github.com:llvm/llvm-project
> Downloaded 260.95 MiB | 9.14 MiB/s
> Took 19.94 seconds
```
Another option is to provide a date based reduction of the history to retrieve everything newer than a provided time point.
```bash
$ # This command was executed on 08.11.2025.
$ time git clone --shallow-since=01.10.2025 git@github.com:llvm/llvm-project
> Downloaded 293.63 MiB | 10.42 MiB/s
> Took 28.86 seconds
```
Implicitly, these commands downloaded _only_ the `main` branch and the repository knows nothing about other branches!
Adding `--no-single-branch` option to the clone provides all branches.
```bash
$ time git clone --depth=1 --no-single-branch git@github.com:llvm/llvm-project
> Downloaded 1.38 GiB | 10.96 MiB/s
> Took 212.92 seconds
```
For big repositories this is undesirable.
The section about [Managing References](#managing-references) provides tips on how to find a middle ground.
Increasing the history length is described in [its own section](#deepen-the-git-history)

## Partial and Sparse Clones

Another optimization available to `git` is to retrieve the actual objects only on demand.
This is best combined with `git-sparse-checkout`, as most big repositories are monorepos and only fractions of it are actually necessary to retrieve for daily work.
Note, that the following command _does not limit_ the history.
```bash
$ # Option 1: Recommended for CI.
$ time git clone --filter=tree:0 git@github.com:llvm/llvm-project
> Downloaded in total 270 MiB | 6.56 MiB/s
> Took 58.24 seconds

$ # Option 2: Recommended for Developers
$ time git clone --filter=blob:none git@github.com:llvm/llvm-project
```
After applying the `--filter` option, `git` defers the actual download of objects to the point of use, usually a `checkout` of a commit.
If objects are missing at point of use, they are downloaded for the `checkout`.

Combining the fetch-filter with `--sparse` and history reduction leads to an incredibly fast clone.
```bash
$ time git clone \
            --filter=blob:none \
            --depth=1 \
            --sparse \
            git@github.com:llvm/llvm-project
> Downloaded 5.56 MiB | 4.77 MiB/s
> Took 0.56 seconds
```
To get a useful repository, the checkout must be increased in scope causing follow up downloads.
```bash
$ cd llvm-project
$ time git sparse-checkout add clang llvm
> Downloaded 175.24 MiB | 10.31 MiB/s
> Took 25.40 seconds
```
Even though the clone itself was fast, there is some later cost.

## Using Alternates

Because `git` is a content addressable filesystem, object access is safe combining multiple locations or repositories.
This is not obvious first, but well explained in the [GitBook](https://git-scm.org/book) under Chapter 10 and `man gitrepository-layout` under `alternates`.
Exploiting this property allows Github to serve so many forks of a repository without massive overhead.

When cloning, it is possible to reference an _existing_ object storage on your local drive and `git` will lookup objects from there, too.
This comes in extremely handy for CI environments that perform containerized builds _from scratch_ in every run.
A worker can maintain a repository mirror that is regularly updated (e.g. with [git-mirror.sh - shameless plug](https://github.com/JonasToth/git-mirror.sh)).
The mirror is mounted into the CI run's container, performing _clean clones_ with the additional `--reference` parameter to the mirror.
All objects found don't need to travel via network and the overall operation becomes incredibly fast.
```bash
$ time git clone \
            --filter=blob:none \
            --depth=1 \
            --sparse \
            --reference llvm-project/ \
            git@github.com:llvm/llvm-project \
            llvm-project-recloned
> Downloaded 0.0MiB!
> Took 0.37 seconds
$ cd llvm-project-recloned
$ git sparse-checkout add llvm clang
> Downloaded 2.30 KiB
> Took 3.45 seconds
```
The 2.3KiB of data is a new commit that was not present in the previous clone of LLVM.

Note, that the referenced repository is in best case created and synced with `git clone --mirror`.
The required network traffic and therefore load on the repository hoster is reduced drastically if frequent `git clone` operations are performed.

Adding alternates manually requires adding the path to the `objects/` directory to either the file `.git/objects/info/alternates` or to the environment variable `GIT_ALTERNATE_OBJECT_DIRECTORIES`.

## Using Worktrees

Another important aspect of developing on a large (mono)repo is the aspect on working on multiple branches at a time.
Switching branches leads to recompilation and other annoying issues.
Sometimes it is simply better to have the repository available in multiple directories and the desired branches per directory.
Instead of cloning the repository multiple times, even if its an optimized clone, one should use `git-worktree` instead.

Creating a worktree by hand is effectively adding a new clone with an `alternate` and reusing the objects from a different repository.
The builtin command `git worktree add` is just way simpler.
A second worktree knows the connection to the original repository and has better integration, so no there is no need to hack it together using the alternate mechanism.
Using `scalar` for cloning will already prepare a workspace that is structured to manage multiple worktree ergonomically.

## Managing References

The previous sections showed that content retrieval cost is strongly correlated with the amount of objects your repository holds -- who would have guessed that.
History reduction and additional scope reduction of your checkout reduce the number of objects in your repository to a minimum.

If the repository is used for software development the reduction to a single branch (induced by `--depth=1` or similar) on the other hand is too impractical.
Having _all_ branches of a big repository is overwhelming.
The middle ground is requires managing the references of your repository and having a bit of naming hygiene in your project.
Take a look at the file `llvm-project/.git/config` from one of the shallow clones.
```toml
[core]
    repositoryformatversion = 1
    filemode = true
    bare = false
    logallrefupdates = true
[remote "origin"]
    url = git@github.com:llvm/llvm-project
    fetch = +refs/heads/main:refs/remotes/origin/main
    promisor = true
    partialclonefilter = blob:none
[branch "main"]
    remote = origin
    merge = refs/heads/main
[extensions]
    worktreeConfig = true
```
The section `[remote "origin"]` contains a single `fetch =` line.
Only the `main` branch is synchronized with `origin`.

The branch-scope can be widened to _all_ branches using the following command.
```bash
$ git remote set-branches origin "*"
$ grep -A 4 'remote "origin"' .git/config
> [remote "origin"]
>     url = git@github.com:llvm/llvm-project
>     promisor = true
>     partialclonefilter = blob:none
>     fetch = +refs/heads/*:refs/remotes/origin/*
```
Now all branches would be synchronized and a follow-up `git fetch` starts to download _a lot_.

Using a naming scheme for branches allows you to limit the number of branches you synchronize.
In my work environment there are `develop/*` branches that contain feature development
and `merge/*` branches for automated release maintenance to down/up-merge development branches.
The `releases/*` branches are the target branches.
Additionally, the maintained releases are part of the branch name.
Adding an identifier for your own branches can work, but might be to cumbersone.

I am only interested in the `develop/*` branches, as these are relevant for code review or for helping out, and the `releases/*`.
All other references are _not_ synchronized with the following example config.
```toml
fetch = +refs/heads/master:refs/remotes/origin/master
fetch = +refs/heads/releases/*:refs/remotes/origin/releases/*
fetch = +refs/heads/develop/*:refs/remotes/origin/develop/*
```
Taking only the latest developments for version `R25` for example would look like this:
```toml
fetch = +refs/heads/master:refs/remotes/origin/master
fetch = +refs/heads/releases/*:refs/remotes/origin/releases/*
fetch = +refs/heads/develop/R25/*:refs/remotes/origin/develop/R25/*
```
It is possible to remap references, but this is not in scope of this blog post.

## Deepen the History

Converting a repository to a full history is done with `git fetch --unshallow`.
Extending the shallow history works either based on a commit count with `git fetch --deepen=1000` or via a time point with `git fetch --shallow-since=02.05.2025`.

## General Speed Improvements

The following quick notes provide pointers to look out for.
- again, use `scalar`, it controls the right knobs out of the box
- call `git maintenance start` on your repository
- consider using `git config core.fsMonitor true` and `git config core.untrackedCache true`
- have your repository on fast storage, at least SSD, maybe even NVME drives
- use the `git sparse-checkout --cone` mode
- use a fast filesystem - I noticed drastic differences between native Linux/ext4 and Windows/ntfs
- regularly rebase your developments to keep the repository-view similar between your used branches, avoiding large index-updates.

## References

- `man git-clone`
- `man git-fetch`
- `man gitrepository-layout`
- [GitBook explaining Objects and Trees](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Github Post about Partial Clones](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/)
