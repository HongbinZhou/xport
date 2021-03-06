========
Xport
========

Python reader for SAS XPORT data transport files (``*.xpt``).



What's it for?
==============

XPORT is the binary file format used by a bunch of `United States
government agencies`_ for publishing data sets. It made a lot of sense
if you were trying to read data files on your IBM mainframe back in
1988.

The official `SAS specification for XPORT`_ is relatively
straightforward. The hardest part is converting IBM-format floating
point to IEEE-format, which the specification explains in detail.

There was an `update to the XPT specification`_ for SAS v8 and above.
This module *has not yet been updated* to work with the new version.
However, if you're using SAS v8+, you're probably not using XPT
format. The changes to the format appear to be trivial changes to the
metadata, but this module's current error-checking will raise a
``ValueError``.

.. _United States government agencies: https://www.google.com/search?q=site:.gov+xpt+file

.. _SAS specification for XPORT: http://support.sas.com/techsup/technote/ts140.pdf

.. _update to the XPT specification: https://support.sas.com/techsup/technote/ts140_2.pdf



Reading XPT
===========

This module mimics the ``json`` and ``pickle`` modules of the standard
library, providing ``load`` and ``loads`` functions for reading data
from a file-like object and from a string object, respectively.

.. code:: python

    with open('example.xpt', 'rb') as f:
        rows = xport.load(f)



Each row will be a namedtuple, with an attribute for each field in the
dataset. Values in the row will be either a unicode string or a float,
as specified by the XPT file metadata. Note that since XPT files are
in an unusual binary format, you should open them using mode ``'rb'``.



The ``load`` and ``loads`` functions can also return the data as
columns rather than rows. The columns will be an ``OrderedDict``
mapping the column labels as strings to the column values as lists of
either strings or floats.

.. code:: python

    with open('example.xpt', 'rb') as f:
        mapping = xport.load(f, mode='columns')



This module also offers reading behavior similar to the standard
library ``csv`` module, for iterating over the data one row at a time,
rather than loading all rows at once. Note that ``xport.Reader`` is
capitalized, unlike ``csv.reader``.

.. code:: python

    with open('example.xpt', 'rb') as f:
        for row in xport.Reader(f):
            print row



For convenient conversion to a `NumPy`_ array or `Pandas`_ dataframe,
you can use ``to_numpy`` and ``to_dataframe``.

.. code:: python

    a = xport.to_numpy('example.xpt')
    df = xport.to_dataframe('example.xpt')

.. _NumPy: http://www.numpy.org/

.. _Pandas: http://pandas.pydata.org/



The ``Reader`` object has a handful of metadata attributes:

* ``Reader.fields`` -- Names of the fields in each observation.

* ``Reader.version`` -- SAS version number used to create the XPT file.

* ``Reader.os`` -- Operating system used to create the XPT file.

* ``Reader.created`` -- Date and time that the XPT file was created.

* ``Reader.modified`` -- Date and time that the XPT file was last modified.



You can also use the ``xport`` module as a command-line tool to convert an XPT
file to CSV (comma-separated values) file.::

    $ python -m xport example.xpt > example.csv



If you want to access specific records, you should use the ``load``
function to gather the rows in a list or use one of ``itertools``
recipes_ for quickly consuming and throwing away unncessary elements.

.. code:: python

    # Collect all the records in a list for random access
    rows = load(f)

    # Select only record 42
    from itertools import islice
    row = next(islice(xport.Reader(f), 42, None))

    # Select only the last 42 records
    from collections import deque
    rows = deque(xport.Reader(f), maxlen=42)

.. _recipes: https://docs.python.org/2/library/itertools.html#recipes



Writing XPT
===========

Similarly to the ``json`` and ``pickle`` modules, ``xport`` provides
``dump`` (to file) and ``dumps`` (to string) functions to transform
Python objects into XPT file format.

The ``dump`` and ``dumps`` functions are in ``'rows'`` mode by default
and expect an iterable of iterables, like a list of tuples. In this
case, the column labels have not been specified and will automatically
be assigned as 'x0', 'x1', 'x2', ..., 'xM'.

.. code:: python

    rows = [('a', 1), ('b', 2)]
    with open('example.xpt', 'wb') as f:
        dump(f, rows)



To specify the column labels in ``'rows'`` mode, each row can be a
mapping (such as a ``dict``) of the column labels to that row's
values. Each row should have the same keys. Passing in rows as
namedtuples, or any instance of a ``tuple`` that has a ``._fields``
attribute, will set the column labels to the attribute names of the
first row.

.. code:: python

    rows = [{'letters': 'a', 'numbers': 1}, {'letters': 'b', 'numbers': 2}]
    with open('example.xpt', 'wb') as f:
        dump(f, rows)



The ``'columns'`` mode expects a mapping of labels (as string) to
columns (as iterable) or an iterable of (label, column) pairs.

.. code:: python

    mapping = {'numbers': [1, 3.14, 42],
               'text': ['life', 'universe', 'everything']}

    # as a mapping of labels to columns
    with open('answers.xpt', 'wb') as f:
        dump(f, mapping, mode='columns')

    # as an iterable of (label, column) pairs
    with open('answers.xpt', 'wb') as f:
        dump(f, mapping.items(), mode='columns')



Column labels are restricted to 40 characters. Column names are
restricted to 8 characters and will be automatically created based on
the column label -- the first 8 characters, non-alphabet characters
replaced with underscores, padded to 8 characters if necessary. All
text strings, including column labels, will be converted to bytes
using the ISO-8859-1 encoding. Any byte strings will not be changed
and may create invalid XPT files if they were encoded inappropriately.

Unfortunately, writing XPT files cannot cleanly mimic the ``csv``
module, because we must examine all rows before writing any rows to
correctly write the XPT file headers.



Recent changes
==============

* Added capability to write XPT files

* Added ``load`` and ``loads`` functions to match the new ``dump`` and
  ``dumps`` functions


Authors
=======

Original version by `Jack Cushman`_, 2012.
Major revision by Michael Selik, 2016.

.. _Jack Cushman: https://github.com/jcushman

