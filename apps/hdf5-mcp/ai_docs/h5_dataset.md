Title: Datasets — h5py 3.14.0 documentation        

*   Datasets
*   View page source

Datasets
=========

Datasets are very similar to NumPy arrays. They are homogeneous collections of data elements, with an immutable datatype and (hyper)rectangular shape. Unlike NumPy arrays, they support a variety of transparent storage features such as compression, error-detection, and chunked I/O.

They are represented in h5py by a thin proxy class which supports familiar NumPy operations like slicing, along with a variety of descriptive attributes:

> *   **shape** attribute
>     
> *   **size** attribute
>     
> *   **ndim** attribute
>     
> *   **dtype** attribute
>     
> *   **nbytes** attribute
>     

h5py supports most NumPy dtypes, and uses the same character codes (e.g. `'f'`, `'i8'`) and dtype machinery as Numpy. See FAQ for the list of dtypes h5py supports.

Creating datasets
------------------

New datasets are created using either `Group.create_dataset()` or `Group.require_dataset()`. Existing datasets should be retrieved using the group indexing syntax (`dset = group["name"]`).

To initialise a dataset, all you have to do is specify a name, shape, and optionally the data type (defaults to `'f'`):

\>>> dset \= f.create\_dataset("default", (100,))
\>>> dset \= f.create\_dataset("ints", (100,), dtype\='i8')

Note

This is not the same as creating an Empty dataset.

You may also initialize the dataset to an existing NumPy array by providing the data parameter:

\>>> arr \= np.arange(100)
\>>> dset \= f.create\_dataset("init", data\=arr)

Assigning an array into a group works like specifying `data` and no other parameters:

\>>> f\["init"\] \= arr

Keywords `shape` and `dtype` may be specified along with `data`; if so, they will override `data.shape` and `data.dtype`. It’s required that (1) the total number of points in `shape` match the total number of points in `data.shape`, and that (2) it’s possible to cast `data.dtype` to the requested `dtype`.

Reading & writing data
-----------------------

HDF5 datasets reuse the NumPy slicing syntax to read and write to the file. Slice specifications are translated directly to HDF5 “hyperslab” selections, and are a fast and efficient way to access data in the file. The following slicing arguments are recognized:

> *   Indices: anything that can be converted to a Python long
>     
> *   Slices (i.e. `[:]` or `[0:10]`)
>     
> *   Field names, in the case of compound data
>     
> *   At most one `Ellipsis` (`...`) object
>     
> *   An empty tuple (`()`) to retrieve all data or scalar data
>     

Here are a few examples (output omitted).

\>>> dset \= f.create\_dataset("MyDataset", (10,10,10), 'f')
\>>> dset\[0,0,0\]
\>>> dset\[0,2:10,1:9:3\]
\>>> dset\[:,::2,5\]
\>>> dset\[0\]
\>>> dset\[1,5\]
\>>> dset\[0,...\]
\>>> dset\[...,6\]
\>>> dset\[()\]

There’s more documentation on what parts of numpy’s fancy indexing are available in h5py.

For compound data, it is advised to separate field names from the numeric slices:

\>>> dset.fields("FieldA")\[:10\]   \# Read a single field
\>>> dset\[:10\]\["FieldA"\]          \# Read all fields, select in NumPy

It is also possible to mix indexing and field names (`dset[:10, "FieldA"]`), but this might be removed in a future version of h5py.

To retrieve the contents of a scalar dataset, you can use the same syntax as in NumPy: `result = dset[()]`. In other words, index into the dataset using an empty tuple.

For simple slicing, broadcasting is supported:

\>>> dset\[0,:,:\] \= np.arange(10)  \# Broadcasts to (10,10)

Broadcasting is implemented using repeated hyperslab selections, and is safe to use with very large target selections. It is supported for the above “simple” (integer, slice and ellipsis) slicing only.

Warning

Currently h5py does not support nested compound types, see GH1197 for more information.

### Multiple indexing

Indexing a dataset once loads a numpy array into memory. If you try to index it twice to write data, you may be surprised that nothing seems to have happened:

\>>> f \= h5py.File('my\_hdf5\_file.h5', 'w')
\>>> dset \= f.create\_dataset("test", (2, 2))
\>>> dset\[0\]\[1\] \= 3.0  \# No effect!
\>>> print(dset\[0\]\[1\])
0.0

The assignment above only modifies the loaded array. It’s equivalent to this:

\>>> new\_array \= dset\[0\]
\>>> new\_array\[1\] \= 3.0
\>>> print(new\_array\[1\])
3.0
\>>> print(dset\[0\]\[1\])
0.0

To write to the dataset, combine the indexes in a single step:

\>>> dset\[0, 1\] \= 3.0
\>>> print(dset\[0, 1\])
3.0

### Length and iteration

As with NumPy arrays, the `len()` of a dataset is the length of the first axis, and iterating over a dataset iterates over the first axis. However, modifications to the yielded data are not recorded in the file. Resizing a dataset while iterating has undefined results.

On 32-bit platforms, `len(dataset)` will fail if the first axis is bigger than 2\*\*32. It’s recommended to use `Dataset.len()` for large datasets.

Chunked storage
----------------

An HDF5 dataset created with the default settings will be contiguous; in other words, laid out on disk in traditional C order. Datasets may also be created using HDF5’s chunked storage layout. This means the dataset is divided up into regularly-sized pieces which are stored haphazardly on disk, and indexed using a B-tree.

Chunked storage makes it possible to resize datasets, and because the data is stored in fixed-size chunks, to use compression filters.

To enable chunked storage, set the keyword `chunks` to a tuple indicating the chunk shape:

\>>> dset \= f.create\_dataset("chunked", (1000, 1000), chunks\=(100, 100))

Data will be read and written in blocks with shape (100,100); for example, the data in `dset[0:100,0:100]` will be stored together in the file, as will the data points in range `dset[400:500, 100:200]`.

Chunking has performance implications. It’s recommended to keep the total size of your chunks between 10 KiB and 1 MiB, larger for larger datasets. Also keep in mind that when any element in a chunk is accessed, the entire chunk is read from disk.

Since picking a chunk shape can be confusing, you can have h5py guess a chunk shape for you:

\>>> dset \= f.create\_dataset("autochunk", (1000, 1000), chunks\=True)

Auto-chunking is also enabled when using compression or `maxshape`, etc., if a chunk shape is not manually specified.

The iter\_chunks method returns an iterator that can be used to perform chunk by chunk reads or writes:

\>>> for s in dset.iter\_chunks():
\>>>     arr \= dset\[s\]  \# get numpy array for chunk

Resizable datasets
-------------------

In HDF5, datasets can be resized once created up to a maximum size, by calling `Dataset.resize()`. You specify this maximum size when creating the dataset, via the keyword `maxshape`:

\>>> dset \= f.create\_dataset("resizable", (10,10), maxshape\=(500, 20))

Any (or all) axes may also be marked as “unlimited”, in which case they may be increased up to the HDF5 per-axis limit of 2\*\*64 elements. Indicate these axes using `None`:

\>>> dset \= f.create\_dataset("unlimited", (10, 10), maxshape\=(None, 10))

For a 1D dataset, `maxshape` can be an integer instead of a tuple. But to make an unlimited 1D dataset, `maxshape` must be a tuple `(None,)`. Passing `None` gives the default behaviour, where the initial size is also the maximum.

Note

Resizing an array with existing data works differently than in NumPy; if any axis shrinks, the data in the missing region is discarded. Data does not “rearrange” itself as it does when resizing a NumPy array.

Filter pipeline
----------------

Chunked data may be transformed by the HDF5 filter pipeline. The most common use is applying transparent compression. Data is compressed on the way to disk, and automatically decompressed when read. Once the dataset is created with a particular compression filter applied, data may be read and written as normal with no special steps required.

Enable compression with the `compression` keyword to `Group.create_dataset()`:

\>>> dset \= f.create\_dataset("zipped", (100, 100), compression\="gzip")

Options for each filter may be specified with `compression_opts`:

\>>> dset \= f.create\_dataset("zipped\_max", (100, 100), compression\="gzip", compression\_opts\=9)

### Lossless compression filters

GZIP filter (`"gzip"`)

Available with every installation of HDF5, so it’s best where portability is required. Good compression, moderate speed. `compression_opts` sets the compression level and may be an integer from 0 to 9, default is 4.

LZF filter (`"lzf"`)

Available with every installation of h5py (C source code also available). Low to moderate compression, very fast. No options.

SZIP filter (`"szip"`)

Patent-encumbered filter used in the NASA community. Not available with all installations of HDF5 due to legal reasons. Consult the HDF5 docs for filter options.

### Custom compression filters

In addition to the compression filters listed above, compression filters can be dynamically loaded by the underlying HDF5 library. This is done by passing a filter number to `Group.create_dataset()` as the `compression` parameter. The `compression_opts` parameter will then be passed to this filter.

See also

hdf5plugin

A Python package of several popular filters, including Blosc, LZ4 and ZFP, for convenient use with h5py

HDF5 Filter Plugins

A collection of filters as a single download from The HDF Group

Registered filter plugins

The index of publicly announced filter plugins

Note

The underlying implementation of the compression filter will have the `H5Z_FLAG_OPTIONAL` flag set. This indicates that if the compression filter doesn’t compress a block while writing, no error will be thrown. The filter will then be skipped when subsequently reading the block.

### Scale-Offset filter

Filters enabled with the `compression` keywords are _lossless_; what comes out of the dataset is exactly what you put in. HDF5 also includes a lossy filter which trades precision for storage space.

Works with integer and floating-point data only. Enable the scale-offset filter by setting `Group.create_dataset()` keyword `scaleoffset` to an integer.

For integer data, this specifies the number of bits to retain. Set to 0 to have HDF5 automatically compute the number of bits required for lossless compression of the chunk. For floating-point data, indicates the number of digits after the decimal point to retain.

Warning

Currently the scale-offset filter does not preserve special float values (i.e. NaN, inf), see https://forum.hdfgroup.org/t/scale-offset-filter-and-special-float-values-nan-infinity/3379 for more information and follow-up.

### Shuffle filter

Block-oriented compressors like GZIP or LZF work better when presented with runs of similar values. Enabling the shuffle filter rearranges the bytes in the chunk and may improve compression ratio. No significant speed penalty, lossless.

Enable by setting `Group.create_dataset()` keyword `shuffle` to True.

### Fletcher32 filter

Adds a checksum to each chunk to detect data corruption. Attempts to read corrupted chunks will fail with an error. No significant speed penalty. Obviously shouldn’t be used with lossy compression filters.

Enable by setting `Group.create_dataset()` keyword `fletcher32` to True.

Multi-Block Selection
----------------------

The full H5Sselect\_hyperslab API is exposed via the MultiBlockSlice object. This takes four elements to define the selection (start, count, stride and block) in contrast to the built-in slice object, which takes three elements. A MultiBlockSlice can be used in place of a slice to select a number of (count) blocks of multiple elements separated by a stride, rather than a set of single elements separated by a step.

For an explanation of how this slicing works, see the HDF5 documentation.

For example:

\>>> dset\[...\]
array(\[ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10\])
\>>> dset\[MultiBlockSlice(start\=1, count\=3, stride\=4, block\=2)\]
array(\[ 1,  2,  5,  6,  9, 10\])

They can be used in multi-dimensional slices alongside any slicing object, including other MultiBlockSlices. For a more complete example of this, see the multiblockslice\_interleave.py example script.

Fancy indexing
---------------

A subset of the NumPy fancy-indexing syntax is supported. Use this with caution, as the underlying HDF5 mechanisms may have different performance than you expect.

For any axis, you can provide an explicit list of points you want; for a dataset with shape (10, 10):

\>>> dset.shape
(10, 10)
\>>> result \= dset\[0, \[1,3,8\]\]
\>>> result.shape
(3,)
\>>> result \= dset\[1:6, \[5,8,9\]\]
\>>> result.shape
(5, 3)

The following restrictions exist:

*   Selection coordinates must be given in increasing order

*   Duplicate selections are ignored

*   Very long lists (> 1000 elements) may produce poor performance

NumPy boolean “mask” arrays can also be used to specify a selection. The result of this operation is a 1-D array with elements arranged in the standard NumPy (C-style) order. Behind the scenes, this generates a laundry list of points to select, so be careful when using it with large masks:

\>>> arr \= numpy.arange(100).reshape((10,10))
\>>> dset \= f.create\_dataset("MyDataset", data\=arr)
\>>> result \= dset\[arr \> 50\]
\>>> result.shape
(49,)

Changed in version 2.10: Selecting using an empty list is now allowed. This returns an array with length 0 in the relevant dimension.

Creating and Reading Empty (or Null) datasets and attributes
-------------------------------------------------------------

HDF5 has the concept of Empty or Null datasets and attributes. These are not the same as an array with a shape of (), or a scalar dataspace in HDF5 terms. Instead, it is a dataset with an associated type, no data, and no shape. In h5py, we represent this as either a dataset with shape `None`, or an instance of `h5py.Empty`. Empty datasets and attributes cannot be sliced.

To create an empty attribute, use `h5py.Empty` as per Attributes:

\>>> obj.attrs\["EmptyAttr"\] \= h5py.Empty("f")

Similarly, reading an empty attribute returns `h5py.Empty`:

\>>> obj.attrs\["EmptyAttr"\]
h5py.Empty(dtype="f")

Empty datasets can be created either by defining a `dtype` but no `shape` in `create_dataset`:

\>>> grp.create\_dataset("EmptyDataset", dtype\="f")

or by `data` to an instance of `h5py.Empty`:

\>>> grp.create\_dataset("EmptyDataset", data\=h5py.Empty("f"))

An empty dataset has shape defined as `None`, which is the best way of determining whether a dataset is empty or not. An empty dataset can be “read” in a similar way to scalar datasets, i.e. if `empty_dataset` is an empty dataset:

\>>> empty\_dataset\[()\]
h5py.Empty(dtype="f")

The dtype of the dataset can be accessed via `<dset>.dtype` as per normal. As empty datasets cannot be sliced, some methods of datasets such as `read_direct` will raise a `TypeError` exception if used on a empty dataset.

Reference
----------

_class_ h5py.Dataset(_identifier_)

Dataset objects are typically created via `Group.create_dataset()`, or by retrieving existing datasets from a file. Call this constructor to create a new Dataset bound to an existing `DatasetID` identifier.

\_\_getitem\_\_(_args_)

NumPy-style slicing to retrieve data. See Reading & writing data.

\_\_setitem\_\_(_args_)

NumPy-style slicing to write data. See Reading & writing data.

\_\_bool\_\_()

Check that the dataset is accessible. A dataset could be inaccessible for several reasons. For instance, the dataset, or the file it belongs to, may have been closed elsewhere.

\>>> f \= h5py.open(filename)
\>>> dset \= f\["MyDS"\]
\>>> f.close()
\>>> if dset:
...     print("dataset accessible")
... else:
...     print("dataset inaccessible")
dataset inaccessible

read\_direct(_array_, _source\_sel\=None_, _dest\_sel\=None_)

Read from an HDF5 dataset directly into a NumPy array, which can avoid making an intermediate copy as happens with slicing. The destination array must be C-contiguous and writable, and must have a datatype to which the source data may be cast. Data type conversion will be carried out on the fly by HDF5.

source\_sel and dest\_sel indicate the range of points in the dataset and destination array respectively. Use the output of `numpy.s_[args]`:

\>>> dset \= f.create\_dataset("dset", (100,), dtype\='int64')
\>>> arr \= np.zeros((100,), dtype\='int32')
\>>> dset.read\_direct(arr, np.s\_\[0:10\], np.s\_\[50:60\])

write\_direct(_source_, _source\_sel\=None_, _dest\_sel\=None_)

Write data directly to HDF5 from a NumPy array. The source array must be C-contiguous. Selections must be the output of numpy.s\_\[<args>\]. Broadcasting is supported for simple indexing.

astype(_dtype_)

Return a read-only view allowing you to read data as a particular type. Conversion is handled by HDF5 directly, on the fly:

\>>> dset \= f.create\_dataset("bigint", (1000,), dtype\='int64')
\>>> out \= dset.astype('int16')\[:500\]
\>>> out.dtype
dtype('int16')

This can be faster than reading the data and then converting it with NumPy:

\>>> out \= dset\[:500\].astype('int16')  \# Typically slower

In case of variable-width strings, calling `.astype('T')` (NumPy’s native variable-width strings) is more efficient than reading the data into an object-type array; read more at NumPy variable-width strings.

Changed in version 3.14: Added support for NumPy variable-width strings (`dtype='T'`).

Changed in version 3.9: `astype()` can no longer be used as a context manager.

asstr(_encoding\=None_, _errors\='strict'_)

Only for string datasets. Returns a wrapper to read data as Python string objects:

\>>> s \= dataset.asstr()\[0\]

encoding and errors work like `bytes.decode()`, but the default encoding is defined by the datatype - ASCII or UTF-8. This is not guaranteed to be correct.

Note

If you don’t require backwards compatibility with NumPy 1.x or h5py <3.14, you should consider reading into NumPy native strings instead, which can be much faster, with `.astype('T')`. Read more at NumPy variable-width strings.

New in version 3.0.

fields(_names_)

Get a wrapper to read a subset of fields from a compound data type:

\>>> 2d\_coords \= dataset.fields(\['x', 'y'\])\[:\]

If names is a string, a single field is extracted, and the resulting arrays will have that dtype. Otherwise, it should be an iterable, and the read data will have a compound dtype.

New in version 3.0.

iter\_chunks()

Iterate over chunks in a chunked dataset. The optional `sel` argument is a slice or tuple of slices that defines the region to be used. If not set, the entire dataspace will be used for the iterator.

For each chunk within the given region, the iterator yields a tuple of slices that gives the intersection of the given chunk with the selection area. This can be used to read or write data in that chunk.

A TypeError will be raised if the dataset is not chunked.

A ValueError will be raised if the selection region is invalid.

New in version 3.0.

resize(_size_, _axis\=None_)

Change the shape of a dataset. size may be a tuple giving the new dataset shape, or an integer giving the new length of the specified axis.

Datasets may be resized only up to `Dataset.maxshape`.

len()

Return the size of the first axis.

make\_scale(_name\=''_)

Make this dataset an HDF5 dimension scale.

You can then attach it to dimensions of other datasets like this:

other\_ds.dims\[0\].attach\_scale(ds)

You can optionally pass a name to associate with this scale.

virtual\_sources()

If this dataset is a virtual dataset, return a list of named tuples: `(vspace, file_name, dset_name, src_space)`, describing which parts of the dataset map to which source datasets. The two ‘space’ members are low-level `SpaceID` objects.

shape

NumPy-style shape tuple giving dataset dimensions.

dtype

NumPy dtype object giving the dataset’s type.

size

Integer giving the total number of elements in the dataset.

nbytes

Integer giving the total number of bytes required to load the full dataset into RAM (i.e. dset\[()\]). This may not be the amount of disk space occupied by the dataset, as datasets may be compressed when written or only partly filled with data. This value also does not include the array overhead, as it only describes the size of the data itself. Thus the real amount of RAM occupied by this dataset may be slightly greater.

New in version 3.0.

ndim

Integer giving the total number of dimensions in the dataset.

maxshape

NumPy-style shape tuple indicating the maximum dimensions up to which the dataset may be resized. Axes with `None` are unlimited.

chunks

Tuple giving the chunk shape, or None if chunked storage is not used. See Chunked storage.

compression

String with the currently applied compression filter, or None if compression is not enabled for this dataset. See Filter pipeline.

compression\_opts

Options for the compression filter. See Filter pipeline.

scaleoffset

Setting for the HDF5 scale-offset filter (integer), or None if scale-offset compression is not used for this dataset. See Scale-Offset filter.

shuffle

Whether the shuffle filter is applied (T/F). See Shuffle filter.

fletcher32

Whether Fletcher32 checksumming is enabled (T/F). See Fletcher32 filter.

fillvalue

Value used when reading uninitialized portions of the dataset, or None if no fill value has been defined, in which case HDF5 will use a type-appropriate default value. Can’t be changed after the dataset is created.

external

If this dataset is stored in one or more external files, this is a list of 3-tuples, like the `external=` parameter to `Group.create_dataset()`. Otherwise, it is `None`.

is\_virtual

True if this dataset is a virtual dataset, otherwise False.

dims

Access to Dimension Scales.

is\_scale

Return `True` if the dataset is also a dimension scale, `False` otherwise.

attrs

Attributes for this dataset.

id

The dataset’s low-level identifier; an instance of `DatasetID`.

ref

An HDF5 object reference pointing to this dataset. See Using object references.

regionref

Proxy object for creating HDF5 region references. See Using region references.

name

String giving the full path to this dataset.

file

`File` instance in which this dataset resides

parent

`Group` instance containing this dataset.
