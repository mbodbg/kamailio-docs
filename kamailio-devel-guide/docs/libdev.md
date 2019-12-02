# Internal Library Development #

Starting with v3.0, &kamailio; has support for internal libraries. They are collection
of C functions to be used by many modules, but not having a general purpose for SIP
server in order to be included in core.

An internal library is automatically loaded at runtime if there is a module in config file that
requires code from it

Among benefits of internal libraries:

* core is smaller - more suitable for embedded devices, as well as it is more stable
overall footprint is smaller - duplicated code in several modules can be collected in
an internal library

* development flexibility - code offering same functionality by a different implementation
can co-exist, allowing to switch and test which one is better, without adding/removing
code from code or modules

## Library Location ##

The library has to be added as a sub-directory of **lib/** folder.
For example the **trie** library is located
in **lib/trie/**.

When adding a new internal library, simply create
a folder for it and place the Makefile, source code and header files inside it.

## Library Makefile ##

The Makefile for internal library specify name, version and external dependencies for it.

    ...
    include ../../Makefile.defs
    auto_gen=
    NAME:=trie
    MAJOR_VER=1
    MINOR_VER=0
    BUGFIX_VER=0
    LIBS=
    
    include ../../Makefile.libs
    ...

The above example is from trie library, which will result on Linux in an object file libtrie.1.0.0.so.

Other Makefile variables such as DEFS or SER_LIBS can be used for libraries in the same manner as for
modules. For example, an internal library can depend on another internal library - simply add the
dependency to SER_LIBS variable.

## Library Source Code ##

The source code and headers have to be placed in files inside library's directory. There is no
real restriction of what you can have inside, besides valid C code. however, it is recommended
to use a naming pattern for C functions and global variables that will reduce the risk of naming
conflicts.

A good practice is to declare only static global variables and export functions that give access
(read/write) to them. Exported functions (non-static functions) should be prefixed by a token
that tries to suggest the library and build an unique name. If you look inside trie library, the
**lib/tree/dtree.c** file, all the functions are prefixed with
**dtree_**.

To use a library, include the files with the prototypes of the functions that you need in your
module and then use them where you need. In the **Makefile** of the
module, you have to add the dependency on the respective library by setting accordingly the
variable **SER_LIBS**. See
**Module Development** section for more details on this topic.
