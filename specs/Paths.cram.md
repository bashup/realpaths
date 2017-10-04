## dirname and basename

realpath.dirname and realpath.basename should produce the same results as their operating system counterparts.

    $ @echo() { "${@:2}" && echo "$REPLY"; }
    $ @assert() { "${@:2}" && [[ $REPLY == "$1" ]] || echo "${*:2}: expected '$1', got '$REPLY'"; }
    $ @compare() { @assert "$("$1" "$2")" realpath.$1 "$2"; }
    $ @check() { @compare basename "$1"; @compare dirname "$1"; }
    $ source "$TESTDIR/../realpaths"

    $ for p1 in '' / // . ..; do
    > for p2 in '' /; do
    > for p3 in foo . ..; do
    > for p4 in '' /bar /. /..; do
    > for p5 in '' / // /// /. /..; do
    > @check "$p1$p2$p3$p4$p5"
    > done; done; done; done; done

## realpath.join

Outputs PWD with no args, joins relative args to PWD, removes empty parts, and canonicalizes ./..:

    $ @assert "$PWD" realpath.join
    $ @assert "$PWD" realpath.join .
    $ @assert "$PWD/x" realpath.join x
    $ @assert "$PWD/x/y" realpath.join x y
    $ @assert "$PWD/x/y" realpath.join x/z ../y
    $ @assert "$PWD/x/y/z" realpath.join x y//z
    $ @assert "$PWD/x/y/z" realpath.join x y '' z
    $ @assert "$PWD/x/y/z" realpath.join x y '' z/.
    $ @assert "$PWD/x/y/z" realpath.join x y '' z/. q .. r .. .. z

Ignores arguments to the left of an absolute path:

    $ @assert /etc realpath.join /etc
    $ @assert /etc/z/xq realpath.join x y z /etc/z q/../xq


## realpath.resolve

Returns first non-symlink:

    $ @assert "x" realpath.resolve x
    $ ln -s y x
    $ @assert "y" realpath.resolve x
    $ ln -s z y
    $ @assert "z" realpath.resolve x

or last symlink before recursion sets in:

    $ ln -s x z
    $ @assert "z" realpath.resolve x
    $ @assert "y" realpath.resolve z
    $ @assert "x" realpath.resolve y

and relative-pathed symlinks are normalized to the symlink directory

    $ mkdir q
    $ @assert "q/../z" realpath.resolve q/../x
    $ rm x y z; rmdir q

## realpath.canonical

Current directory:

    $ @assert "$PWD" realpath.canonical .
    $ @assert "$PWD" realpath.canonical "$PWD"


Nonexistent file:

    $ @assert "$PWD/x" realpath.canonical x
    $ @assert "$PWD/x" realpath.canonical "$PWD/x"


Symlink to non-existent target:

    $ ln -s y x
    $ @assert "$PWD/y" realpath.canonical x
    $ @assert "$PWD/y" realpath.canonical "$PWD/x"


Symlink chain:

    $ ln -s z y
    $ @assert "$PWD/z" realpath.canonical x
    $ @assert "$PWD/z" realpath.canonical "$PWD/x"


Symlink loop breaks on the looper:

    $ ln -s y z
    $ @assert "$PWD/z" realpath.canonical x


Absolute links and directories:

    $ rm x y z
    $ touch y
    $ mkdir z
    $ ln -s $PWD/y x
    $ ln -s $PWD/x z/x

    $ @assert "$PWD/y" realpath.canonical x
    $ @assert "$PWD/y" realpath.canonical z/x

    $ ln -sf ../x z/x
    $ @assert "$PWD/y" realpath.canonical x
    $ @assert "$PWD/y" realpath.canonical z/x

    $ ln -s z q
    $ @assert "$PWD/z" realpath.canonical q
    $ @assert "$PWD/y" realpath.canonical q/../x
    $ @assert "$PWD/z/a" realpath.canonical q/a

### realpath.location

Returns the absolute (but not canonical) location of the directory physically containing its target.  Is equivalent to `realpath.join` if target isn't a symlink:

    $ @assert "$PWD"       realpath.location x
    $ @assert "$PWD"       realpath.location z
    $ @assert "$PWD/z"     realpath.location z/a
    $ @assert "$PWD"       realpath.location q/x  # -> ../x
    $ @assert "$PWD/q"     realpath.location q/foo
    $ @assert "$PWD"       realpath.location q/../foo
    $ @assert "$PWD/q/foo" realpath.location q/foo/bar

### Missing elements

    $ @assert "$PWD/z/foo/bar/baz" realpath.canonical  q/foo/bar/baz
    $ @assert "$PWD/z/foo/bar"     realpath.canonical  q/foo/bar
    $ @assert /non-existent        realpath.canonical /non-existent
    $ @assert /etc/non-existent    realpath.canonical /etc/non-existent

### Subdirectories, accessible and inaccessible

Works while the directory is accesible:

    $ mkdir foo
    $ cd foo
    $ @assert "$OLDPWD/z" realpath.canonical "$OLDPWD/q"
    $ @assert "$OLDPWD/y" realpath.canonical "$OLDPWD/q/x"
    $ @assert "$OLDPWD/y" realpath.canonical "../q/x"

We can follow symlinks even if $PWD is inaccessible:

    $ rmdir ../foo
    $ @assert "$OLDPWD/z" realpath.canonical "$OLDPWD/q"
    $ @assert "$OLDPWD/y" realpath.canonical "../q/x"

And we can do non-symlink canonicalizing on the base dir of the results:

    $ @assert "$OLDPWD/z" realpath.canonical "../z"
    $ @assert "$OLDPWD/z/foo" realpath.canonical "../z/foo"
    $ @assert "$OLDPWD/foo" realpath.canonical "../foo"
    $ @assert "$OLDPWD/z" realpath.canonical "../q"
