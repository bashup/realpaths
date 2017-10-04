## Fast path manipulation and symlink resolution for bash

Sadly, it's hard to find a portable way to get the "real path" of a file, and shell-based emulations of `realpath ` and `readlink -f` are usually quite slow compared to the real thing, and don't return *quite* the same results.  (Dealing with missing elements, symlink loops, and the like can also be a challenge, even with the "real" `readlink` or `realpath`!)

So, this module is a thorough implementation of symlink resolution and path canonicalization using plain, portable `readlink` with no options.  Its symlink resolution is about as fast as `readlink -f` when there is only one symlink to follow, and *enormously* faster when the target isn't a symlink.  (It's slower on symlink chains, though.)

To avoid the need for subshells, all of this module's functions output a single path result via the bash `REPLY` variable.  All functions return success, *always*, as they are designed to provide meaningful results even in the presence of missing, looping, or inaccessible files, symlinks, or directories.

Most of this module's functions also return absolute, well-formed paths: that is, paths that begin with `/` and contain no `.`, `..` or empty components.  (The exceptions are `realpath.follow`, `realpath.dirname`, and `realpath.basename`, which can return relative paths.)

(**Note:** unlike their operating system counterparts, these functions do *not* accept options, and always return exactly **one** path as a result.  Do not pass `--` to them or expect to get multiple results for multiple arguments!)

### Installation, Requirements And Use

Copy and paste the [code](realpaths) into your script, or place it on `PATH` and `source "$(command -v realpaths)"`.  You can install it on your `PATH` with [basher](https://github.com/basherpm/basher), using `basher install bashup/realpaths`.  The code is licensed CC0, so you are not required to add any attribution or copyright notices to your project.

The code's only extenal requirement is `readlink`, which is used only to resolve a single symlink level, and therefore can be a GNU, BSD, or OS X readlink implementation.

### Resolving Symlinks

#### realpath.location

Sets `REPLY` to the absolute, well-formed path of a physical directory that contains (or *would* contain) the supplied path.  (Equivalent to the `dirname` of  `realpath.resolved "$1"`.) Always succeeds.

(Pass this function `"$BASH_SOURCE"` to get the directory of the currently-executing file, or `"$0"` to get the directory of the current main script.)

#### realpath.resolved

Sets `REPLY` to the absolute, well-formed path of `$1`, with symlinks in the final path portion resolved.  (Equivalent to the `realpath.absolute` of `realpath.follow "$1"`.)  Always succeeds.  The result is not a symlink unless it is inaccessible or part of a symlink cycle.

#### realpath.follow

Sets `REPLY` to the first non-symlink (or last accessible, non-looping symlink) in the symlink chain beginning at `$1`.  Replies with the unchanged `$1` if it is not a symlink.  Always succeeds.  The result is not a symlink unless it is inaccessible or part of a symlink cycle.  (Note: the result *can* be a relative path, if both `$1` and all the symlinks in the chain are relative.  Use `realpath.resolved` instead if you want a guaranteed-absolute path.)

### Path String Manipulation

#### realpath.absolute *[paths...]*

Sets `REPLY` to the absolute, well-formed combination of the supplied path(s). Always succeeds.

Each path may be absolute or relative.  The resulting path is the combination of the *last* absolute path in the list supplied, combined with any relative paths that follow it.  If no absolute paths are given, the relative paths are processed relative to `$PWD` -- so passing zero arguments simply returns `$PWD`.

Relative path parts are resolved *logically* rather than physically.  That is to say, `..` is processed by removing elements from the path string, rather than by inspecting the filesystem.  (So symlinks are not processed in any way, and the existence or accessibility of the files and directories is irrelevant: with the exception of defaulting to `$PWD`, the result is obtained solely via string manipulation of the supplied paths.)

#### realpath.relative *path [basedir]*

Sets `REPLY` to the shortest relative path from *basedir* to *path*.  *basedir* defaults to `$PWD` if not supplied.  Always succeeds.

The *path* and *basedir* are preprocessed with `realpath.absolute`, so they can be relative paths, or absolute ones containing relative components.  (The result is identical to calling Python's `os.path.relpath` function with the same arguments, but is much faster than even the fork to start Python would be.)

The main use case for this function is portably creating relative symlinks without needing `ln --relative` or `realpath --relative-to`.  Specifically, if you are doing `ln -s somepath somedir/link`, you can make it relative using `realpath.relative somepath somedir; ln -s "$REPLY" somedir/link`.

#### realpath.dirname

Sets `REPLY` to the directory name of `$1`, always returning success.  Produces the *exact* same results as `REPLY=$(dirname -- "$1")` except much, *much* faster.  Always succeeds.

#### realpath.basename

Sets `REPLY` to the basename of `$1`.  Produces the *exact* same results as `REPLY=$(basename -- "$1")` except much, *much* faster.  Always succeeds.

### Determinining Canonical Paths

#### realpath.canonical

Sets `REPLY` to the *fully canonicalized* form of `$1`, resolving symlinks in every part of the path where that can be done, roughly equivalent to `realpath -m` or `readlink -m`.   Always succeeds, but potentially quite slow, depending on how many directories are symlinks.

You don't really need this function unless you are trying to determine whether divergent paths lead to the "same" file: for use cases that don't involve comparing paths,  `realpath.absolute` should be sufficient.  (Note, too, that using canonical paths can result in user confusion, since they have to reconcile their inputs with your outputs.)

## License

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license" href="http://creativecommons.org/publicdomain/zero/1.0/"><img src="https://licensebuttons.net/p/zero/1.0/80x15.png" style="border-style: none;" alt="CC0" /></a><br />
  To the extent possible under law, <a rel="dct:publisher" href="https://github.com/pjeby"><span property="dct:title">PJ Eby</span></a>
  has waived all copyright and related or neighboring rights to <span property="dct:title">bashup/realpaths</span>.
This work is published from: <span property="vcard:Country" datatype="dct:ISO3166" content="US" about="https://github.com/bashup/realpaths">United States</span>.
</p>