## Fast path manipulation and symlink resolution for bash

Sadly, it's hard to find a portable way to get the "real path" of a file, and shell-based emulations of `realpath ` and `readlink -f` are usually quite slow compared to the real thing, and don't return *quite* the same results.  (Dealing with missing elements, symlink loops, and the like can also be a challenge, even with the "real" `readlink` or `realpath`!)

So, this module is a thorough implementation of symlink resolution and path canonicalization using plain `readlink` with no options.  It is about as fast as `readlink -f` when there is only one symlink to follow, and *enormously* faster when the target isn't a symlink.  (It's slower on symlink chains, though.)

All of this module's functions output their results to the bash `REPLY` variable: this is done to avoid the need for subshells.  All functions return success, *always*, as they are designed to provide meaningful results even in the presence of missing, looping, or inaccessible files, symlinks, or directories.

Most of this module's functions also return absolute, well-formed paths: that is, paths that begin with `/` and contain no `.`, `..` or empty components.  (The exceptions are `realpath.resolve`, `realpath.dirname`, and `realpath.basename`, which can return relative paths.)

(**Note:** unlike their operating system counterparts, these functions do *not* accept multiple arguments (except for `realpath.join`), and do *not* take option parameters.  Do not pass `--` to them or expect to get multiple results for multiple arguments!)

### Resolving Symlinks

#### realpath.location

Sets `REPLY` to the absolute, well-formed path of a physical directory that contains (or *would* contain) the supplied path.  (Equivalent to the parent directory of  `realpath.absolute "$1"`.) Always succeeds.

(Pass this function  `"$BASH_SOURCE"` to get the directory of the currently-executing file, or `"$0"` to get the directory of the current main script.)

#### realpath.absolute

Sets `REPLY` to the absolute, well-formed path of `$1` with symlinks in the final path portion removed.  (Equivalent to the `realpath.join` of `realpath.resolve "$1"`.)  Always succeeds.

#### realpath.resolve

Sets `REPLY` to the first non-symlink (or last non-looping symlink) in the symlink chain beginning at `$1`.  Replies with the unchanged `$1` if it is not a symlink.  Always succeeds.  (Note: the result can be a relative path, if both the argument and its symlinks are relative.  Use `realpath.absolute` instead if you want an absolute path.)

### Path Manipulation

#### realpath.join *parts...*

Join zero or more path components, setting `REPLY` to an absolute, well-formed result. Always succeeds.

Arguments to the left of an absolute path argument are discarded (similar to Python's `os.path.join`).  If there are no absolute path arguments, the absolute path is computed based on the current `$PWD`.

All path components are resolved *logically* rather than physically: that is to say, `..` is processed by removing elements from the path string rather than by inspecting the filesystem.  (So symlinks are not processed in any way, and the existence or accessibility of the files and directories is irrelevant.)

#### realpath.dirname *path*

Sets `REPLY` to the directory name of *path*, always returning success.  Produces the *exact* same results as `REPLY=$(dirname -- "$1")` except much, *much* faster.  Always succeeds.

#### realpath.basename *path*

Sets `REPLY` to the basename of *path*.  Produces the *exact* same results as `REPLY=$(basename -- "$1")` except much, *much* faster.

### Determinining Canonical Paths

#### realpath.canonical *path*

Sets `REPLY` to the *fully canonicalized* form of path, resolving symlinks in every part of the path where that can be done, roughly equivalent to `realpath -m` or `readlink -m`.   Always succeeds, but potentially quite slow, depending on how many directories are symlinks.

You don't really need this function unless you are trying to determine whether divergent paths lead to the "same" file: for use cases that don't involve comparing paths,  `realpath.absolute` should be sufficient.  (Note, too, that using canonical paths can result in user confusion, since they have to reconcile their inputs with your outputs.)

