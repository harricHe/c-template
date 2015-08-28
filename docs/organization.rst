.. _organization:

Project organization
====================

This section describes the expected layout of directories and files in a
project that uses this template.  It also describes some of the features
that are automatically set up for you by the template.


Directory layout overview
-------------------------

The following is an overview of the important directories in the project
template:

*include*
   Should contain any public header files for your project

*src*
   Should contain C source code for any libraries and command-line
   utilities in your project

*docs*
   Should contain Sphinx documentation for your project

*tests*
   Should contain your project's test suite


Public include files
--------------------

If your project provides a C library, you should place the public header
files for your library into the project's *include* directory.  The
CMake build scripts will automatically find and install any *\*.h* file
in this directory.  Any directory structure that you create underneath
*include* will be reproduced when installing into ``${PREFIX}``.  So, if
your project contains the following files::

    include/coollib.h
    include/coollib/basics.h
    include/coollib/extras.h
    include/coollib/io.h

Then your ``make install`` target will install the following::

    ${PREFIX}/include/coollib.h
    ${PREFIX}/include/coollib/basics.h
    ${PREFIX}/include/coollib/extras.h
    ${PREFIX}/include/coollib/io.h


Source code
-----------

The *src* directory should contain one subdirectory for each library and
command-line program in your project.  Individual source files should
along belong to a single library or program.  (If you need to share code
across multiple programs, but don't want to include it in an installed
shared library, you can create a “helper” static library, which is
linked into each program, but which isn't installed on its own.)

You should also provide a pkg-config file for each public shared library
in your project.  This pkg-config file contains details that other users
will use to set up the include and linker paths when using the library.
(If your project contains several public shared libraries, each one
should get its own pkg-config file.)

The default *src/CMakeLists.txt* file contains skeleton CMake rules for
building a shared library and installing its corresponding pkg-config
file.  This file makes heavy use of some helper macros::

    add_c_library(
        [name]
        OUTPUT_NAME [lib_name]
        PKGCONFIG_NAME [pkgconfig_name]
        VERSION [lib_version]
        SOURCES [sources]
        LIBRARIES [prereqs]
        LOCAL_LIBRARIES [local_prereqs]
    )

defines a new library, which can be built either shared or static as requested
by the person running the build.  ``[name]`` is the local CMake identifier that
you use in other parts of the build scripts to refer to this library.
``[lib_name]`` is the "actual" name of the library, as it will be created in the
local filesystem.  (This should not include any ``lib`` prefix or ``.so``/etc
suffix.)  ``[pkgconfig_name]`` is the name of the pkgconfig file for this
library — it should not include the ``.pc`` suffix, and you must have a
``src/[pkgconfig_name].pc.in`` file which will be used to construct the final
pkgconfig description.  ``[lib_version]`` is the `library version`_ of the
library (note that this is **not** the same as the project's version).
``[sources]`` is a list of the C files that should be compiled to produce this
library.  ``[prereqs]`` is a list of prerequisite libraries that this library
should be linked with.  (See below for how to declare which prereqs are
available in this project.)  ``[local_prereqs]`` are a list of other *local*
libraries (i.e., those that are a part of this project) that this library should
be linked with.  (They should already have been defined via another
``add_c_library`` call.)

.. _library version: http://www.gnu.org/software/libtool/manual/html_node/Updating-version-info.html#Updating-version-info


::

    add_c_executable(
        [name]
        [SKIP_INSTALL]
        OUTPUT_NAME [prog_name]
        SOURCES [sources]
        LIBRARIES [prereqs]
        LOCAL_LIBRARIES [local_prereqs]
    )

defines a new command-line program.  ``[SKIP_INSTALL]``, if given, tells CMake
not to install the program.  (This is useful if you need it for some
command-line test cases, for instance.)  All of the other parameters have the
same meaning as in ``add_c_library``.


Both of the above commands have a ``LIBRARIES`` clause that lets you specify
upstream dependencies that your libraries or programs depend on.  You must
**declare** each of these dependencies before you can use them when defining
your libraries and programs.  The prereqs are declared in the "Check for
prerequisite libraries" section of the top-level *CMakeLists.txt* file.  You
have a few options available::

    pkgconfig_prereq([name])
    pkgconfig_prereq([name]>=[version])
    library_prereq([name])

Ideally, your upstream dependencies will use pkgconfig, in which case you can
use the ``pkgconfig_prereq`` macro.  This will look for a pkgconfig file with
the given ``[name]``, and use the contents of that file to tell CMake which
physically library file to link with.  If you need a particular version, you can
you the ``>=[version]`` syntax.  In both cases, ``[name]`` is then a valid value
to use in a ``LIBRARIES`` clause when defining a library or program.

If the upstream library doesn't provide a pkgconfig file, you must use the
``library_prereq`` macro.  This uses CMake's ``find_library`` command to look
for a library with the given name, and if it's found, then makes ``[name]``
available as a valid value to use in a ``LIBRARIES`` clause when defining a
library or program.


Documentation
-------------

The *docs* directory should contain a collection of specially formatted Markdown
files, which `pandoc`_ will use to create man pages for your project.

.. _pandoc: http://pandoc.org/

The default *docs/CMakeLists.txt* file includes comments explaining how to
define which man pages you've written.  The macros take care of rebuilding the
man pages if the Markdown source has changed at all.

The resulting man pages are also checked into the git repository, so that you
only need to install `pandoc`_ if you edit the Markdown source.  If you haven't
done that, the CMake build scripts will install the existing compiled man pages
at install time.


Tests
-----

You should implement test cases for every library and program in your
project.  Without exception!  Seriously, write some test cases.

The template supports two kinds of test cases: unit tests based on the
`check`_ library, and command-line tests that let you check the stdout
and stderr of an arbitrary command.  (Presumably, you'll use the first
to check the individual functions and types of a shared library, and the
second to test the overall behavior of a command-line program.)

.. _check: http://check.sourceforge.net/

Library tests
~~~~~~~~~~~~~

To write a library test, you create a new :file:`test-{something}.c`
file in the *tests* directory, and add a call to the ``add_c_test`` macro
in *tests/CMakeLists.txt*.  The default template links these test programs with
the ``check`` library, and with any library that's part of your project.  (You
shouldn't need to explicitly mention third-party libraries needed by the
project's libraries; CMake will automatically include any transitive
dependencies that it can determine.)

With these definitions in place, the ``make test`` target in your
project's Makefile will automatically run each of the test programs that
you've defined.  If a test fails, you can run it directly to see its
output::

    $ tests/test-something

(This should be run from the top-level build directory.)

Often you'll have several test programs that share common code.  A
useful strategy is to create a *tests/helpers.h* file to contain this
common code, and to include this file in each of your test programs.

Command-line tests
~~~~~~~~~~~~~~~~~~

You can also write test cases that execute an arbitrary command, and
verify the contents of its stdout and stderr streams.  (You can also
provide an arbitrary stdin stream that will be passed in to the
program.)  We use `cram`_ for these; the default CMake build scripts will
automatically find any ``*.t`` files in the *tests* directory, and use `cram`_
to run those as part of ``make test``.

.. _cram: https://pypi.python.org/pypi/cram
