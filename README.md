
## Template for building packages for OmniOS

This is an empty framework for building packages for OmniOS.

## Getting Started

Rename `lib/site.sh.template` as `site.sh` and edit it to your taste.
`site.sh` lets you override the build defaults defined in `lib/config.sh`.

Packages will install into their own directory under `$PREFIX` with their
binaries and man pages linked into `${PREFIX}/bin` etc. `$PREFIX` defaults to
`/opt`. If you wish to install elsewhere, modify `$PREFIX`.

### Source Archives

The scripts expect to fetch every package's source code from the same
location. Set up an HTTP server or local directory, and assign it to the
`MIRROR` variable in `site.sh`. If you use a directory, define it as
`/path/to/directory` -- don't use `file://`.

Each program must have its own subdirectory, named like the base name of the
source archive. For instance, `ImageMagick/ImageMagick-7.0.9-3.tar.xz`. The
SHA of the file should be stored alongside it, with the suffix `.sha256`.

If you need to, you can generate the checksum file with a command like

```
$ digest -a sha256 program-1.2.3.tar.xz >program-1.2.3.tar.xz.sha256
```

### Target Repository

The final step in the build process publishes the new package to a repository.
The easiest way to get started is to publish to a directory. The `PKGSRVR`
variable points to this repository.

## Building a Package

### Getting Ready

Create a subdirectory in `build/`. This directory must be the name of the
package it describes, but without a version. For instance, `ImageMagick` or
`apache-maven`. It can contain the following files.

### `build.sh`

This script must be present and executable. It builds a package by setting
variables and calling functions from `lib/functions.sh`. Here is a simple
example.

```
. ../../lib/functions.sh          # Defines the functions which do the work

PROG=example                      # Name of software to be packaged
VER=1.2.3                         # Version of software to be packaged
VERHUMAN=$VER                     # A "friendly" version number, normally the
same as $VER
PKG=vendor/category/example       # What your package will be called
SUMMARY="an example program"      # Briefly, what the software is
DESC="for illustration"           # A more detailed description of the software

set_arch 64                       # Force a 64-bit build

OPREFIX=$PREFIX                   # The install root. $PREFIX is set in site.sh
PREFIX+="/$PROG"                  # Where the package will actually install. It
                                  # will most likely have its binaries linked
                                  # under $OPREFIX

BUILD_DEPENDS_IPS="               # Any packages necessary for compilation
    ooce/library/libpng
"

XFORM_ARGS="                      # Arguments passed to pkgmogrify
    -DPREFIX=${PREFIX#/}
    -DOPREFIX=${OPREFIX#/}
    -DPROG=$PROG
"

CONFIGURE_OPTS="                  # Arguments to ./configure
    --prefix=$PREFIX
    --without-perl
"

init                              # Runs preliminary checks
prep_build                        # Sets up the build space and sets some vars
download_source $PROG $PROG $VER  # Fetches, verifies and unpacks the source
patch_source                      # Applies patches, if any
build                             # configures, makes, and installs the
                                  # software in temporary directory
make_package                      # builds an IPS package from the temporary
                                  # install
clean_up                          # remove the build space and temp install
```

It is well worth reading through `lib/functions.sh` to see what helpers are
available.

Autotools, `cmake`, and Meson builds are supported. If you need some other
build chain, you are free to add your own code to `build.sh`.

### `local.mog`

This must be present. It is used by `pkgmogrify(1)` to generate a final
package by applying transformations and adding instructions to the package
manifest.

`local.mog` should include a `license` line. This is of the form

```
license <file> license=<type>
```

`<file>` is the license file in the source archive, relative to its unpacked
root. `<type>` is the name of the license, which must be referenced in
`doc/license`. (See below.) If you do not wish to include a license file, put
the license name in the `SKIP_LICENCES` variable.

Refer to the `pkgmogrify` man page for more information.

### `doc/license`

This is a tab-separated file. The first field is the type of license, as
referred to in `local.mog`. The second is a string or `nawk`-compatible
regular-expression which uniquely identifies the license. The build process
uses it to check you have licensed your package correctly and to populate the
package metadata with license information.

### `files/`

If you need to add extra files into your packages, for instance SMF manifests,
store them in here.

### `patches/`

To patch the source archive before compiling, put diff files in here, along
with a `series` file which lists the order in which the diffs should be
applied.

### Building and Debugging

To make a package, run your `build.sh` script. All output will go to
`./build.log`. This will help with most problems, but should you need more
information (for instance `config.log`) you can find the working directory
under `/tmp/build_<username>`.

## Gotchas

`pkg-config` probably won't find libraries installed under `/opt/ooce`. You'll
have to tell it where to look by forcing `PKG_CONFIG_PATH` and `LDFLAGS`.

These (and other) build environment variables such should be *appended* to,
not simply *set*. This implies a leading separator. For instance:

```
PKG_CONFIG_PATH+=":/opt/ooce/lib/pkgconfig"
LDFLAGS+=" -L/opt/ooce/lib -R/opt/ooce/lib"
```
