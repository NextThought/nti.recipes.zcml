===================
 Supported options
===================

The recipe supports installing multiple different sets
of ZCML slugs in multiple different output directories.
These sets are specified in grouped options, where ``X``
is the common prefix shared by all options in the group.

X_zcml
    A list of zcml entires. Required.

    format::

        zcml := package ":" filename
        package := dottedname | dottedname "-" ( include_name )

    The ``filename`` is the fully specified file, such as
    ``browser.zcml``, whereas the ``include_name`` is a relative
    portion mising the ``.zcml`` extension, defaulting to
    ``configure`` (it was originally validated against a strict list
    of possibilities, but that is no longer the case). If the filename
    is not given, the ``include_name`` is used.

    .. note:: Even if you just want to specify ``X_features``, you
			  must still have an entry named ``X_zcml``. It may be
			  empty in that case.

X_location
    A directory name relative to the etc-directory
    to put the generated slugs in. Required.

X_file
    A convenient shortuct if all or most of the zcml entries would
    have the same ``include_name``. Set this option to make it the
    default instead of configure. Optional.

X_features
    If this optional directive is provided, it is a space and newline
    separated list of ZCML features that should be provided when the
    output directory is processed. These are provided in the first
    file.

There are two global options:

deployment
    The name of a ``zc.recipe.deployment`` part containing the
    directory definitions. We will use the ``etc-directory`` defined
    in this part as the base for locations.

etc-directory
    If you do not specify a ``deployment``, then this value will
    be used as the etc-directory.



Example usage
=============

We'll start by creating a buildout that uses the recipe. We will list
three packages that we'd like to create slugs for (a ``*`` is ignored
for backwards compatibility), along with a set of features we want to
be provided::

    >>> write('buildout.cfg',
    ... """
    ... [buildout]
    ... parts = test1
    ...
    ... [test1]
    ... recipe = nti.recipes.zcml
    ... etc-directory = ${buildout:directory}/zope/etc
    ... package_location = package-includes
    ... package_features = foo bar
    ...                    baz
    ... package_zcml =
    ...     *
    ...     my.package
    ...     somefile:my.otherpackage
    ...     my.thirdpackage-meta
    ... """)

Running the buildout gives us::

    >>> print(system(buildout))
    Installing test1.
    While:
      Installing test1.
    Error: The parents of '/.../sample-buildout/zope/etc/package-includes' do not exist

We need to have a valid etc directory. Let's create one::

    >>> mkdir("zope")
    >>> mkdir("zope", "etc")
    >>> print(system(buildout))
    Installing test1.

We now have a package include directory::

    >>> ls("zope", "etc")
    d  package-includes

It does contain ZCML slugs::

    >>> ls("zope", "etc", "package-includes")
    -  000-features.zcml
    -  001-my.package-configure.zcml
    -  002-somefile-configure.zcml
    -  003-my.thirdpackage-meta.zcml

These files contain the usual stuff::

    >>> cat("zope", "etc", "package-includes", "000-features.zcml")
    <configure xmlns="http://namespaces.zope.org/zope" xmlns:meta="http://namespaces.zope.org/meta">
        <meta:provides feature="foo" />
        <meta:provides feature="bar" />
        <meta:provides feature="baz" />
    </configure>
    >>> cat("zope", "etc", "package-includes", "001-my.package-configure.zcml")
    <include package="my.package" file="configure.zcml" />
    >>> cat("zope", "etc", "package-includes", "002-somefile-configure.zcml")
    <include package="somefile" file="my.otherpackage" />
    >>> cat("zope", "etc", "package-includes", "003-my.thirdpackage-meta.zcml")
    <include package="my.thirdpackage" file="meta.zcml" />


Error and Corner Cases
======================

Now we will discuss how various corner cases and errors are handled.

No ZCML and No Features
-----------------------

If you do not specify any ZCML or features, no files are generated
(note that we're using a new part name, causing the old part to be
uninstalled)::


    >>> write('buildout.cfg',
    ... """
    ... [buildout]
    ... parts = zcml
    ...
    ... [zcml]
    ... recipe = nti.recipes.zcml
    ... etc-directory = ${buildout:directory}/zope/etc
    ... package_location = empty-includes
    ... package_features =
    ... package_zcml =
    ... """)

    >>> print(system(buildout))
    Uninstalling test1.
    Installing zcml.
    <BLANKLINE>


No directory is created for this part, and when the old part was
uninstalled, it left behind its directory, but no files::

    >>> ls("zope", "etc")
    d  package-includes

    >>> ls("zope", "etc", "package-includes")

Using a Deployment Reference
============================

As mentioned above, we can use a ``zc.recipe.deployment`` section to
find the ``etc`` directory (in reality, we can accept any part that
has an ``etc-directory`` setting); this will override any locally
specified ``etc-directory``. We haven't created the directory we
specified (and we're not using ``zc.recipe.deployment`` to
automatically do so) so this buildout will fail::

    >>> write('buildout.cfg',
    ... """
    ... [buildout]
    ... parts = zcml
    ...
    ... [deployment-settings]
    ... etc-directory = ${buildout:directory}/zope/deployment-etc
    ...
    ... [zcml]
    ... recipe = nti.recipes.zcml
    ... deployment = deployment-settings
    ... etc-directory = ${buildout:directory}/zope/etc
    ... package_location = empty-includes
    ... package_features = foo
    ... package_zcml =
    ... """)

    >>> print(system(buildout))
    Uninstalling zcml.
    Installing zcml.
    While:
      Installing zcml.
    Error: The parents of '/.../zope/deployment-etc/empty-includes' do not exist
    <BLANKLINE>

Malformed Package Names
=======================

An error is raised if the package name is malformed (although at the
moment only the most egregious violations are detected)::


    >>> write('buildout.cfg',
    ... """
    ... [buildout]
    ... parts = zcml
    ...
    ... [zcml]
    ... recipe = nti.recipes.zcml
    ... etc-directory = ${buildout:directory}/zope/etc
    ... package_location = empty-includes
    ... package_zcml = $not_valid
    ... """)

    >>> print(system(buildout))
    Installing zcml.
    While:
      Installing zcml.
    Error: Invalid package name: '$not_valid' parsed as '$not_valid'
    <BLANKLINE>

Specifying Filenames Twice
==========================

We can specify both the ``include_name`` and the ``filename`` for a
single entry::

    >>> write('buildout.cfg',
    ... """
    ... [buildout]
    ... parts = zcml
    ...
    ... [zcml]
    ... recipe = nti.recipes.zcml
    ... etc-directory = ${buildout:directory}/zope/etc
    ... package_location = package-includes
    ... package_zcml = my.package-foo:filename.zcml
    ... """)

    >>> print(system(buildout))
    Installing zcml.


    >>> ls("zope", "etc", "package-includes")
    -  001-my.package-foo.zcml

    >>> cat("zope", "etc", "package-includes", "001-my.package-foo.zcml")
    <include package="my.package" file="filename.zcml" />
