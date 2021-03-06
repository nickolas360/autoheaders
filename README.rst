autoheaders
===========

Version 0.3.2

autoheaders automatically generates header files from C source code.


Installation
------------

From the Git repository
~~~~~~~~~~~~~~~~~~~~~~~

Clone the repository (you’ll need to have `Git`_ installed)::

    git clone https://git.nickolas360.com/nickolas360/autoheaders
    cd autoheaders

Then install with `pip`_::

    sudo pip3 install .

Alternatively, you can run: [1]_ ::

    sudo ./setup.py install

With either command, to install locally, run without ``sudo`` and add the
``--user`` option. [2]_

Run without installing
~~~~~~~~~~~~~~~~~~~~~~

Run the first set of commands in the previous section to clone the repository.
Then install the required dependencies::

    sudo pip3 install -r requirements.txt

To install the dependencies locally, run without ``sudo`` and add the
``--user`` option.

.. _Git: https://git-scm.com


Usage
-----

If you installed autoheaders, you can simply run ``autoheaders``. [3]_
Otherwise, run ``./autoheaders.py``. This will display usage information
similar to the following::

    autoheaders [options] [--] <c-file>
    autoheaders -h | --help | --version

**Arguments:**

* ``<c-file>``:
  The C source code file from which to generate the header. If ``-`` is passed,
  standard input is read (unless the argument is preceded by ``--``).

**Options:**

* ``-p --private``:
  Generate a private header file containing static declarations.

* ``-o <file>``
  Write the header file to the specified file. If given after ``-p``, a private
  header is written. This option may be given twice: once before ``-p``, and
  once after. If not given, the header is written to standard output.

* ``-c <cpp-arg>``:
  Pass arguments to the C preprocessor. Separate arguments with commas, or
  provide multiple ``-c`` options. Use ``\`` to escape characters. [4]_
  *Note that when the preprocessor is run, the current working directory is the
  parent directory of the C file.* It is therefore recommended to convert paths
  in ``-c`` arguments to absolute paths.

* ``--debug``:
  Run the program in debug mode. Exception tracebacks are shown.

The generated header file is written to standard output.

The C preprocessor command that is used is determined by trying the
following options in the order listed:

* The value of the ``AUTOHEADERS_CPP`` environment variable. This is parsed and
  interpreted as a shell command. For example: ``AUTOHEADERS_CPP="gcc -E"``
* ``gcc -E``
* ``clang -E``

The C preprocessor must be compatible with `GCC`_’s preprocessor (``gcc -E``).

See the next section for how to structure your C code so that headers can be
generated properly.

.. _GCC: https://gcc.gnu.org/


Header generation
-----------------

autoheaders parses the given C file and looks for all function definitions,
global variable definitions, and non-``extern`` global variable declarations
(which are essentially zero-initialized definitions).

Definitions and declarations marked ``static`` are ignored. The remaining
function definitions are transformed into function declarations and are added
to the header file. The remaining variable definitions and declarations are
transformed into ``extern`` variable declarations and are added to the header
file.

Additionally, you can explicitly specify code to be added to the header—this is
necessary if you define structs or need certain files to be to be included in
the header. All code between ``#ifdef HEADER`` and ``#endif`` is copied to the
header file, intermixed with the generated declarations (depending on where the
``#ifdef HEADER`` blocks occur in the source file).

Include guards can also be generated. If a comment of the form
``@guard <name>`` is present, an include guard will be generated using the
macro ``<name>``. The comment must appear at the top level, outside of any
blocks or preprocessor conditionals.

Files that are included (with ``#include``) by the C file do not need to exist
and are not processed by autoheaders, *except* for files inside
``#ifdef HEADER`` blocks. [5]_

Sometimes, however, certain ``#include`` statements do need to be processed for
autoheaders to parse the file properly, especially if the included files define
macros that are used at the top level (i.e., not inside functions) by the
original C file. In this case, a comment of the form ``@include`` can be placed
after the ``#include <...>``, *on the same line*. For example:

.. code:: c

    #include <assert.h> /* @include */
    #include <stdio.h>

will cause autoheaders to include ``assert.h`` during its processing, but
``stdio.h`` will not be included (unless these ``#include`` statements appear
in an ``#ifdef HEADER`` block [5]_).


Private headers
---------------

In addition to the public header files normally generated by autoheaders,
private header files can be generated as well. These header files are designed
to be included only by the corresponding C file and remove the need for
forward declarations of static functions.

To generate a private header file, provide the option ``-p``.

``#ifdef HEADER`` blocks will not be included in the private header. To include
code in the private header (for things like private structures), use
``#ifdef PRIVATE_HEADER`` blocks (also closed with ``#endif``, of course).

Finally, the macro ``ANY_HEADER`` will be defined for both public and private
headers, which allows you to use ``#ifdef ANY_HEADER`` blocks to include code
in both headers. You shouldn’t usually need to do this, however.


Example
-------

*Also see the* |example/|_ *directory for a more complete example.*

.. |example/| replace:: *example/*
.. _example/: example/

If the following code is in ``test.c``:

.. code:: c

    // @guard TEST_H

    #include "test.h"
    #include "test.priv.h"
    #include <stdio.h>

    #ifdef HEADER
        #include <stdint.h>

        typedef struct {
            int32_t first;
            int32_t second
        } IntPair;
    #endif

    const IntPair zero_pair = { 0, 0 };

    // Adds a pair of integers.
    int32_t add_pair(IntPair pair) {
        return add(pair.first, pair.second);
    }

    // Adds two integers.
    static int32_t add(int32_t first, int32_t second) {
        printf("Adding %"PRId32" and %"PRId32"\n", first, second);
        return first + second;
    }

then you can run ``autoheaders test.c -o test.h`` to generate the public header
file. ``test.h`` will then contain the following code:

.. code:: c

    #ifndef TEST_H
    #define TEST_H

    #include <stdint.h>

    typedef struct {
        int32_t first;
        int32_t second;
    } IntPair;

    extern const IntPair zero_pair;

    // Adds a pair of integers.
    int32_t add_pair(IntPair pair);

    #endif

Similarly, you can run ``autoheaders test.c -p -o test.priv.h`` to generate the
private header file. ``test.priv.h`` will then contain the following code:

.. code:: c

    // Adds two integers.
    static int32_t add(int32_t first, int32_t second);

You can also generate both the public and private headers at the same time,
which is faster than generating each individually, by running::

    autoheaders test.c -o test.h -p -o test.priv.h

See the `example/`_ directory for a more complete example.

.. _example/: example/


Fake headers
------------

If an included header contains a large about of code, it can cause autoheaders
to run slowly. Certain non-standard headers may not even be able to parse. In
these cases, you can create fake headers that override the real ones when
autoheaders runs.

Fake headers simply need to declare the types and macros from the real header
that your code uses. The types do not need to match the real ones; they just
need to be declared. The recommended way to do this is with typedefs. For
example, ``typedef int div_t;`` is a suitable definition of ``div_t``,
regardless of whether or not ``div_t`` is actually an integer.

Macros used by your code must be defined in the fake header as well. While, as
with types, the fake header macros don’t need to match the real ones, a little
more care must be taken to ensure that the fake macros produce syntactically
valid code.

For example, a fake header for ``pthread.h`` could contain the following:

.. code:: c

    typedef int pthread_t;
    typedef int pthread_mutex_t;
    #define PTHREAD_MUTEX_INITIALIZER 0

Put your fake headers in a directory with a structure that matches that of the
real headers. For example, using the directory ``fake/``, if your code
contained ``#include <pthread.h>``, the fake header would be stored in
``fake/pthread.h``. If your code contained ``#include <pthread/pthread.h>``,
the fake header would be stored in ``fake/pthread/pthread.h``.

After creating your fake headers, you can run autoheaders as follows::

    autoheaders <c-file> -c -I<fake-header-dir>

where ``<fake-header-dir>`` is the directory containing the fake headers.
Following the examples above, autoheaders might be invoked as::

    autoheaders file.c -c -Ifake/

Additionally, you can include your fake header directory automatically by
giving it a special name. When running, autoheaders will look for a directory
named ``.fake-headers/`` in the directory containing the C file or in any parent
directory. If such a directory is found, it will be included with ``-I``.

See `this article about pycparser`__ for more information about fake headers.

__ https://eli.thegreenplace.net/2015/on-parsing-c-type-declarations-and-fake-headers


Troubleshooting
---------------

The most likely error to be encountered is when code contains non-standard C
extensions; for example, ``__attribute__`` in GCC. C code is parsed after
preprocessing, so the use of non-standard features in any included files causes
problems for the parser.

These issues can be easily mitigated by modifying `shim.h`_.
(``__attribute__`` and some other extensions are currently handled and do not
cause errors.) ``shim.h`` contains typedefs and macro definitions that
transform the code into standards-compliant C (at least enough to be parsed).
For more information, see `this article about pycparser`__.

.. _shim.h: autoheaders/shim.h
__ https://eli.thegreenplace.net/2015/on-parsing-c-type-declarations-and-fake-headers

If you find that something is missing from ``shim.h``, please file an issue or
open a pull request.


Dependencies
------------

* `Python`_ ≥ 3.5 with `pip`_ installed
* A `GCC`_-compatible compiler (specifically, a compatible C preprocessor);
  see the `Usage`_ section.
* The following Python packages (the installation instructions above handle
  installing these):

  - `pycparser`_ ≥ 2.18
  - `setuptools`_ [6]_ ≥ 39.0.0

.. _Python: https://www.python.org/
.. _GCC: https://gcc.gnu.org/
.. _Usage: #usage
.. _pycparser: https://pypi.python.org/pypi/pycparser/


What’s new
----------

Version 0.3.2:

* Array declarations without explicit sizes (appearing as part of a definition)
  are no longer copied to header files, as this causes compilation errors.

Version 0.3.1:

* Fixed issue where array declarations would be ignored.

Version 0.3.0:

* Public and private headers can now be generated at the same time.

Version 0.2.1:

* Clarified how the current working directory changes when the preprocessor is
  run.

Version 0.2.0:

* The order of ``#ifdef HEADER`` blocks and definitions is now preserved.
  If an ``#ifdef HEADER`` block appears after a function definition, it will
  now appear after the generated declaration in the header file.


License
-------

autoheaders is licensed under the GNU General Public License, version 3 or any
later version. See `LICENSE`_. [7]_

This README file has been released to the public domain using `CC0`_.

.. _LICENSE: LICENSE
.. _CC0: https://creativecommons.org/publicdomain/zero/1.0/

.. [1] `setuptools`_ must be installed before running ``setup.py``. If `pip`_
   is installed, ``setuptools`` likely is as well; otherwise, run
   ``sudo pip3 install setuptools`` or ``pip3 install --user setuptools``.

.. [2] If using ``setup.py``, add the ``--user`` option after ``install``
   (rather than before it).

.. [3] If Python package binary directories are not in your ``$PATH``, you may
   have to run ``python3 -m autoheaders`` instead.

.. [4] Backslashes can be used to include commas in the passed arguments: for
   example, ``-c 'arg\,with\,commas'`` will pass the single argument
   ``arg,with,commas`` to the preprocessor. Other backslash escapes are simply
   interpreted as the second character: ``-c 'a\bc\\d'`` becomes ``abc\d``.

.. [5] Including ``#ifdef PRIVATE_HEADER`` and ``#ifdef ANY_HEADER`` blocks.

.. [6] Specifically, ``pkg_resources`` must be installed. Some package managers
   distribute ``pkg_resources`` separately from ``setuptools``. For example,
   in Debian GNU/Linux and many derivatives, ``pkg_resources`` is available
   via ``apt`` in ``python3-pkg-resources``.

.. [7] This does not apply to generated header files; the copyright and license
   status of such files is unaffected by autoheaders.

.. _pip: https://pip.pypa.io
.. _setuptools: https://pypi.org/project/setuptools/
