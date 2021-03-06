Create Dask DataFrames
----------------------

From CSV files
~~~~~~~~~~~~~~

``dask.dataframe.read_csv`` uses ``pandas.read_csv`` and so inherits all of
that functions options.  Additionally it gains two new functionalities

1.  You can provide a globstring

.. code-block:: python

   >>> df = dd.read_csv('data.*.csv.gz', compression='gzip')

2.  You can specify the size of each block of data in bytes of uncompressed
    data.  Note that, especially for text data the size on disk may be much
    less than the number of bytes in memory.

.. code-block:: python

   >>> df = dd.read_csv('data.csv', chunkbytes=10000000)  # 1MB chunks

3.  You can ask to categorize your result.  This is slightly faster at read_csv
    time because we can selectively read the object dtype columns first.  This
    requires a full read of the dataset and may take some time

.. code-block:: python

   >>> df = dd.read_csv('data.csv', categorize=True)


so needs a docstring. Maybe we should have ``iris.csv`` somewhere in
the project.

From an Array
~~~~~~~~~~~~~

You can create a DataFrame from any sliceable array like object including both
NumPy arrays and HDF5 datasets.

.. code-block:: Python

   >>> dd.from_array(x, chunksize=1000000)

From BColz
~~~~~~~~~~

BColz_ is an on-disk, chunked, compressed, column-store.  These attributes make
it very attractive for dask.dataframe which can operate particularly well on
it.  There is a special ``from_bcolz`` function.

.. code-block:: Python

   >>> df = dd.from_bcolz('myfile.bcolz', chunksize=1000000)

In particular column access on a dask.dataframe backed by a ``bcolz.ctable``
will only read the necessary columns from disk.  This can provide dramatic
performance improvements.

From Castra
~~~~~~~~~~~

Castra_ is a tiny, experimental partitioned on-disk data structure designed to
fit the ``dask.dataframe`` model.  It provides columnstore access and range
queries along the index.  It is also a very small project (roughly 400 lines).

.. code-block:: Python

   >>> from castra import Castra
   >>> c = Castra(path='/my/castra/file')
   >>> df = c.to_dask()


From Raw Dask Graphs
~~~~~~~~~~~~~~~~~~~~

This section is for developer information and discusses internal API.  You
should never need to create a dataframe object by hand.

To construct a DataFrame manually from a dask graph you need the following
information:

1.  dask: a dask graph with keys like ``{(name, 0): ..., (name, 1): ...}`` as
    well as any other tasks on which those tasks depend.  The tasks
    corresponding to ``(name, i)`` should produce ``pandas.DataFrame`` objects
    that correspond to the columns and divisions information discussed below.
2.  name: The special name used above
3.  columns: A list of column names
4.  divisions: A list of index values that separate the different partitions.
    Alternatively, if you don't know the divisions (this is common) you can
    provide a list of ``[None, None, None, ...]`` with as many partitions as
    you have plus one.  For more information see the Partitions section in the
    :doc:`dataframe documentation <dataframe>`.

As an example, we build a DataFrame manually that reads several CSV files that
have a datetime index separated by day.  Note, you should never do this.  The
``dd.read_csv`` function does this for you.

.. code-block:: Python

   dsk = {('mydf', 0): (pd.read_csv, 'data/2000-01-01.csv'),
          ('mydf', 1): (pd.read_csv, 'data/2000-01-02.csv'),
          ('mydf', 2): (pd.read_csv, 'data/2000-01-03.csv')}
   name = 'mydf'
   columns = ['price', 'name', 'id']
   divisions = [Timestamp('2000-01-01 00:00:00'),
                Timestamp('2000-01-02 00:00:00'),
                Timestamp('2000-01-03 00:00:00'),
                Timestamp('2000-01-03 23:59:59')]

   df = dd.DataFrame(dsk, name, columns, divisions)


.. _BColz: http://bcolz.blosc.org/
.. _Castra: http://github.com/Blosc/castra
