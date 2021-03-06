---
layout: tactile
title: Creating a binary Debian package
---

In this example, we'll create a binary Emacs Debian package, built from upstream sources.
This guide is a quickstart only, suitable for creating debs for personal use.
The tools we need are ``dpkg -b`` and ``fakeroot``.
I needed to do this because the version of Emacs in Debian repos uses GTK3, which I
dislike because it looks fugly without GNOME 3 themes and I refuse to install GNOME themes
on my i3 setup. I wanted to recompile using GTK2.

* Download the source tarball and unpack.
```
wget http://ftp.gnu.org/pub/gnu/emacs/emacs-24.4.tar.xz
tar xf emacs-24.4.tar.xz
cd emacs-24.4/
```

* Compile locally as any other GNU software.
```
TMPDIR=/tmp/emacs_24.4-bds1_amd64
./configure --prefix="$TMPDIR"/usr/local
make
make install-strip      # Unstripped executables trigger Lintian errors
```
Not that you don't need sudo (or root) access rights to compile and install
to a temporary directory. The ``configure`` script may throw some errors
for missing libraries. Note them down, we need it for a later step. As
mentioned on the [Emacs Wiki][wiki], you need these libs and their development
headers:
```
sudo apt-get install autoconf automake libtool texinfo build-essential \
 xorg-dev libgtk2.0-dev libjpeg-dev libncurses5-dev libdbus-1-dev \
 libgif-dev libtiff-dev libm17n-dev libpng12-dev librsvg2-dev libotf-dev
```

* The ``--prefix`` flag ends in ``/usr/local`` because this is where we want
our compiled bins and libs to go to, after installation of .deb. In a sense,
``$TMPDIR`` acts as the root of the filesystem.

* As mentioned in the [Debian FAQ][faq], the package name must be of the format:
```
<name>_<x.y.z>-<revision-number>_<arch>.deb 
```
which in our case becomes:
```
emacs_24.4-bds1_amd64.deb
```
I use the three letter prefix ``bds`` in revision number so that I can identify 
that it is my package, not the one from stock Debian.

* Compute the checksums of each and every file after ``make install``
```
cd $TMPDIR
find . -type f  -printf '%P\0' | LC_ALL=C sort -z | xargs -r0 md5sum > md5sums
cd -
```

* Create a special folder called ``DEBIAN`` and a ``control`` file in it
```
mkdir $TMPDIR/DEBIAN/             # uppercase DEBIAN mandatory
nano $TMPDIR/DEBIAN/control
mv $TMPDIR/md5sums $TMPDIR/DEBIAN
```

* The contents of the [control file][control], with only the mandatory fields:
```
Package: emacs
Version: 24.4-bds1
Section: editors
Priority: optional
Architecture: amd64
Maintainer: Your Name <email@example.com>
Depends: libc6, libx11, libncurses5, libgtk2.0, libm17n, libotf, libjpeg
Description: GNU Emacs 24.4 using GTK 2.x GUI toolkit.
 This package provides Emacs, the programmable editor.
 The description paragraphs must all be indented by ONE space and when
 the paragraph ends, a full stop must be on its own line, like below.
 .
 More optional description can follow here.
```

* Simple way to get a list of all dependencies is to use ``objdump`` and 
grep for "NEEDED" flag. Combine with ``dpkg -S`` to see which package provides
the NEEDED. With some syntactic sugar:
```
objdump -p $TMPDIR/usr/local/bin/emacs-24.4 | grep NEEDED | \
  awk -F ' ' '{print $2}' | xargs dpkg -S | cut -d':' -f 1 | sort | uniq
```
You can find runtime dependencies using (not reliable?):
```
dpkg-depcheck $TMPDIR/usr/local/bin/emacs-24.4
```

* Now build the package. ``dpkg -b`` is a short alias for ``dpkg-deb --build``.
``dpkg -b`` tunnels the command to ``dpkg-deb --build`` automatically.
```
cd /tmp
fakeroot dpkg -b emacs_24.4-bds1_amd64
```
This takes about 5 minutes or so.

* You can get inspect the build package using:
```
dpkg -I emacs_24.4-bds1_amd64   # same as --info, Note the UPPERCASE I
dpkg -c emacs_24.4-bds1_amd64   # same as --contents
```

* Check that the built package follows Debian standars using a Lint-like tool
called ``lintian``:
```
lintian emacs_24.4-bds1_amd64.deb
```
It shows some warnings and errors like:
```
W: emacs: hardening-no-relro usr/local/bin/ctags
E: emacs: debian-changelog-file-missing
E: emacs: no-copyright-file
E: emacs: dir-in-usr-local usr/local/bin/
E: emacs: file-in-usr-local usr/local/bin/ctags
etc.
```
The first line means our binaries are not [hardened][hardening]. Next,
you must create the files ``copyright``, ``changelog`` and ``changelog.Debian``
in the ``doc`` directory (``$TMPDIR/usr/local/share/doc/emacs/``) and ``gzip -9``
on each file to keep lintian happy. Since this is for personal use, I don't care
so much about these errors. The other error is obvious: we're installing in 
/usr/local which is reserved for end-user and is not supposed to be touched.
(well, I'm the end user here, so it's ok).

See also [this TLDP HOW TO][tldp], [this Ubuntu forum thread][forum], 
[note on conffiles][conf] as well as the [Debian policy manual][dpm].

FAQs:

1. Why not checkinstall ?

This stupid tool *requires* sudo/root rights, even when you pass ``--install=no`` option.

2. What's wrong with the above approach (``dpkg -b``) ?

Packages built like this don't contain fields like integrity checks (md5sum, SHA1, SHA256),
``Size``/``Installed-Size`` of the files, etc. in the ``control`` file. The packages are
not GPG-signed. Therefore, these packages cannot be officially included in APT repos.
It is recommended to use ``dh_make`` and ``dpkg-buildpackage`` instead.

3. Building in a chroot'ed environment ?
You'll come across ``pbuilder`` and ``sbuild``. Go with ``sbuild`` because that is
what is officially used by Debian and Ubuntu. A [comparative quick guide][so].

Debian packaging has **too many** tools which do somewhat similar tasks. An oveview
of some commonly used tools is [here][tools]. No wonder Debian packaging is so
complicated compared to Arch or Fedora.

[How to build a .deb without using any external tools][noext]
Another [quick guide][debm] from a Debian maintainer.
[Expectations of a Debian mentro (aka sponsor)][mentor]
[How to build from .dsc of orphaned Debian package][orph]

[wiki]: http://www.emacswiki.org/emacs/EmacsSnapshotAndDebian
[faq]: https://www.debian.org/doc/manuals/debian-faq/ch-pkg_basics.en.html#s-pkgname
[dmo]: http://deb-multimedia.org/pool/main/a/avidemux/avidemux_2.6.8-dmo3_amd64.deb
[forum]: http://ubuntuforums.org/showthread.php?t=910717
[dpm]: https://www.debian.org/doc/debian-policy/ap-pkg-binarypkg.html
[control]: https://www.debian.org/doc/debian-policy/ch-controlfields.html#s-binarycontrolfiles
[tldp]: http://tldp.org/HOWTO/html_single/Debian-Binary-Package-Building-HOWTO/
[conf]: http://www.electricmonk.nl/log/2011/09/06/creating-simple-debian-packages/
[hardening]: https://wiki.debian.org/Hardening
[tools]: https://www.debian.org/doc/manuals/developers-reference/tools
[so]: http://askubuntu.com/a/199184
[debm]: http://www.pseudorandom.co.uk/2008/sbuild-dm/
[orph]: http://sfxpt.wordpress.com/2013/05/24/debian-packages-building/
[noext]: https://unix.stackexchange.com/questions/30303/how-to-create-a-deb-file-manually
[mentor]: http://vincent.bernat.im/en/debian-package-sponsoring.html
