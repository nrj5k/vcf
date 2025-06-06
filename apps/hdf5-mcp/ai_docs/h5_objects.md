Title: File Objects — h5py 3.14.0 documentation        

*   File Objects
*   View page source

File Objects
=============

File objects serve as your entry point into the world of HDF5. In addition to the File-specific capabilities listed here, every File instance is also an HDF5 group representing the root group of the file.

Opening & creating files
-------------------------

HDF5 files work generally like standard Python file objects. They support standard modes like r/w/a, and should be closed when they are no longer in use. However, there is obviously no concept of “text” vs “binary” mode.

\>>> f \= h5py.File('myfile.hdf5','r')

The file name may be a byte string or unicode string. Valid modes are:

> <table><tbody><tr><td><p>r</p></td><td><p>Readonly, file must exist (default)</p></td></tr><tr><td><p>r+</p></td><td><p>Read/write, file must exist</p></td></tr><tr><td><p>w</p></td><td><p>Create file, truncate if exists</p></td></tr><tr><td><p>w- or x</p></td><td><p>Create file, fail if exists</p></td></tr><tr><td><p>a</p></td><td><p>Read/write if exists, create otherwise</p></td></tr></tbody></table>

Changed in version 3.0: Files are now opened read-only by default. Earlier versions of h5py would pick different modes depending on the presence and permissions of the file.

File drivers
-------------

HDF5 ships with a variety of different low-level drivers, which map the logical HDF5 address space to different storage mechanisms. You can specify which driver you want to use when the file is opened:

\>>> f \= h5py.File('myfile.hdf5', driver\=<driver name\>, <driver\_kwds\>)

For example, the HDF5 “core” driver can be used to create a purely in-memory HDF5 file, optionally written out to disk when it is closed. Here’s a list of supported drivers and their options:

> None
> 
> **Strongly recommended.** Use the standard HDF5 driver appropriate for the current platform. On UNIX, this is the H5FD\_SEC2 driver; on Windows, it is H5FD\_WINDOWS.
> 
> ‘sec2’
> 
> Unbuffered, optimized I/O using standard POSIX functions.
> 
> ‘stdio’
> 
> Buffered I/O using functions from stdio.h.
> 
> ‘core’
> 
> Store and manipulate the data in memory, and optionally write it back out when the file is closed. Using this with an existing file and a reading mode will read the entire file into memory. Keywords:
> 
> backing\_store:
> 
> If True (default), save changes to the real file at the specified path on `close()` or `flush()`. If False, any changes are discarded when the file is closed.
> 
> block\_size:
> 
> Increment (in bytes) by which memory is extended. Default is 64k.
> 
> ‘family’
> 
> Store the file on disk as a series of fixed-length chunks. Useful if the file system doesn’t allow large files. Note: the filename you provide _must_ contain a printf-style integer format code (e.g. %d”), which will be replaced by the file sequence number. Keywords:
> 
> memb\_size: Maximum file size (default is 2\*\*31-1).
> 
> ‘fileobj’
> 
> Store the data in a Python file-like object; see below. This is the default if a file-like object is passed to `File`.
> 
> ‘split’
> 
> Splits the meta data and raw data into separate files. Keywords:
> 
> meta\_ext:
> 
> Metadata filename extension. Default is ‘-m.h5’.
> 
> raw\_ext:
> 
> Raw data filename extension. Default is ‘-r.h5’.
> 
> ‘ros3’
> 
> Enables read-only access to HDF5 files in the AWS S3 or S3-compatible object stores. HDF5 file name must be one of http://, https://, or s3:// resource location. An s3:// location will be translated into an AWS path-style location by h5py. Keywords:
> 
> aws\_region:
> 
> AWS region of the S3 bucket with the file, e.g. `b"us-east-1"`. Default is `b''`. Required for s3:// locations.
> 
> secret\_id:
> 
> AWS access key ID. Default is `b''`.
> 
> secret\_key:
> 
> AWS secret access key. Default is `b''`.
> 
> session\_token:
> 
> AWS temporary session token. Default is `b''`.’ Must be used together with temporary secret\_id and secret\_key. Available from HDF5 1.14.2.
> 
> The argument values must be `bytes` objects. Arguments aws\_region, secret\_id, and secret\_key are required to activate AWS authentication.
> 
> Note
> 
> Pre-built h5py packages on PyPI do not include ros3 driver support. If you want this feature, you could use packages from conda-forge, or build h5py from source against an HDF5 build with ros3. Alternatively, use the file-like object support with a package like s3fs.

In-memory ‘files’
------------------

HDF5 can make a file in memory, without reading or writing a real file. To do this with h5py, use the `File.in_memory()` class method:

\# Start a new HDF5 file in memory
f \= h5py.File.in\_memory()
f\['a'\] \= \[1, 2, 3\]

\# Get the file data as bytes, e.g. to send over the network
f.flush()
hdf\_data \= f.id.get\_file\_image()

\# Turn the bytes back into an h5py File object
f2 \= h5py.File.in\_memory(hdf\_data)

This uses HDF5’s “core” file driver, which is likely to cause fewer odd problems than asking HDF5 to call back into a Python `BytesIO` object (described below).

New in version 3.13.

Python file-like objects
-------------------------

New in version 2.9.

The first argument to `File` may be a Python file-like object, such as an `io.BytesIO` or `tempfile.TemporaryFile` instance. This is a convenient way to create temporary HDF5 files, e.g. for testing or to send over the network.

The file-like object must be open for binary I/O, and must have these methods: `read()` (or `readinto()`), `write()`, `seek()`, `tell()`, `truncate()` and `flush()`.

\>>> tf \= tempfile.TemporaryFile()
\>>> f \= h5py.File(tf, 'w')

Accessing the `File` instance after the underlying file object has been closed will result in undefined behaviour.

When using an in-memory object such as `io.BytesIO`, the data written will take up space in memory. If you want to write large amounts of data, a better option may be to store temporary data on disk using the functions in `tempfile`.

"""Create an HDF5 file in memory and retrieve the raw bytes

This could be used, for instance, in a server producing small HDF5
files on demand.
"""
import io
import h5py

bio \= io.BytesIO()
with h5py.File(bio, 'w') as f:
f\['dataset'\] \= range(10)

data \= bio.getvalue() \# data is a regular Python bytes object.
print("Total size:", len(data))
print("First bytes:", data\[:10\])

Warning

When using a Python file-like object for an HDF5 file, make sure to close the HDF5 file before closing the file object it’s wrapping. If there is an error while trying to close the HDF5 file, segfaults may occur.

Warning

When using a Python file-like object, using service threads to implement the file-like API can lead to process deadlocks.

`h5py` serializes access to low-level hdf5 functions via a global lock. This lock is held when the file-like methods are called and is required to delete/deallocate `h5py` objects. Thus, if cyclic garbage collection is triggered on a service thread the program will deadlock. The service thread can not continue until it acquires the lock, and the thread holding the lock will not release it until the service thread completes its work.

If possible, avoid creating circular references (either via `weakrefs` or manually breaking the cycles) that keep `h5py` objects alive. If this is not possible, manually triggering a garbage collection from the correct thread or temporarily disabling garbage collection may help.

Note

Using a Python file-like object for HDF5 is internally more complex, as the HDF5 C code calls back into Python to access it. It inevitably has more ways to go wrong, and the failures may be less clear when it does. For some common use cases, you can easily avoid it:

*   To create a file in memory and never write it to disk, use the `'core'` driver with `mode='w', backing_store=False` (see File drivers).

*   To use a temporary file securely, make a temporary directory and open a file path inside it.

Version bounding
-----------------

HDF5 has been evolving for many years now. By default, the library will write objects in the most compatible fashion possible, so that older versions will still be able to read files generated by modern programs. However, there can be feature or performance advantages if you are willing to forgo a certain level of backwards compatibility. By using the “libver” option to `File`, you can specify the minimum and maximum sophistication of these structures:

\>>> f \= h5py.File('name.hdf5', libver\='earliest') \# most compatible
\>>> f \= h5py.File('name.hdf5', libver\='latest')   \# most modern

Here “latest” means that HDF5 will always use the newest version of these structures without particular concern for backwards compatibility. The “earliest” option means that HDF5 will make a _best effort_ to be backwards compatible.

The default is “earliest”.

Specifying version bounds has changed from HDF5 version 1.10.2. There are two new compatibility levels: v108 (for HDF5 1.8) and v110 (for HDF5 1.10). This change enables, for example, something like this:

\>>> f \= h5py.File('name.hdf5', libver\=('earliest', 'v108'))

which enforces full backward compatibility up to HDF5 1.8. Using any HDF5 feature that requires a newer format will raise an error.

latest is now an alias to another bound label that represents the latest version. Because of this, the File.libver property will not use latest in its output for HDF5 1.10.2 or later.

Closing files
--------------

If you call `File.close()`, or leave a `with h5py.File(...)` block, the file will be closed and any objects (such as groups or datasets) you have from that file will become unusable. This is equivalent to what HDF5 calls ‘strong’ closing.

If a file object goes out of scope in your Python code, the file will only be closed when there are no remaining objects belonging to it. This is what HDF5 calls ‘weak’ closing.

with h5py.File('f1.h5', 'r') as f1:
ds \= f1\['dataset'\]

\# ERROR - can't access dataset, because f1 is closed:
ds\[0\]

def get\_dataset():
f2 \= h5py.File('f2.h5', 'r')
return f2\['dataset'\]
ds \= get\_dataset()

\# OK - f2 is out of scope, but the dataset reference keeps it open:
ds\[0\]

del ds  \# Now f2.h5 will be closed

User block
-----------

HDF5 allows the user to insert arbitrary data at the beginning of the file, in a reserved space called the user block. The length of the user block must be specified when the file is created. It can be either zero (the default) or a power of two greater than or equal to 512. You can specify the size of the user block when creating a new file, via the `userblock_size` keyword to File; the userblock size of an open file can likewise be queried through the `File.userblock_size` property.

Modifying the user block on an open file is not supported; this is a limitation of the HDF5 library. However, once the file is closed you are free to read and write data at the start of the file, provided your modifications don’t leave the user block region.

Filenames on different systems
-------------------------------

Different operating systems (and different file systems) store filenames with different encodings. Additionally, in Python there are at least two different representations of filenames, as encoded `bytes` or as a Unicode string (`str` on Python 3).

h5py’s high-level interfaces always return filenames as `str`, e.g. `File.filename`. h5py accepts filenames as either `str` or `bytes`. In most cases, using Unicode (`str`) paths is preferred, but there are some caveats.

Note

HDF5 handles filenames as bytes (C `char *`), and the h5py Low-Level API matches this.

### macOS (OSX)

macOS is the simplest system to deal with, it only accepts UTF-8, so using Unicode paths will just work (and should be preferred).

### Linux (and non-macOS Unix)

Filenames on Unix-like systems are natively bytes. By convention, the locale encoding is used to convert to and from unicode; on most modern systems this will be UTF-8 by default (especially since Python 3.7, with **PEP 538**).

Passing Unicode paths will mostly work, and Unicode paths from system functions like `os.listdir()` should always work. But if there are filenames that aren’t in the expected encoding (e.g. on a network filesystem or a removable drive, or because something is misconfigured), you may want to handle them as bytes.

### Windows

Windows systems natively handle filenames as Unicode, and with HDF5 1.10.6 and above filenames passed to h5py as bytes will be used as UTF-8 encoded text, regardless of system configuration.

HDF5 1.10.5 and below could only use filenames with characters from the active code page, e.g. Windows-1252 on many systems configured for European languages. This limitation applies whether you use `str` or `bytes` with h5py.

Chunk cache
------------

Chunked storage allows datasets to be stored on disk in separate pieces. When a part of any one of these pieces is needed, the entire chunk is read into memory before the requested part is copied to the user’s buffer. To the extent possible those chunks are cached in memory, so that if the user requests a different part of a chunk that has already been read, the data can be copied directly from memory rather than reading the file again. The details of a given dataset’s chunks are controlled when creating the dataset, but it is possible to adjust the behavior of the chunk _cache_ when opening the file.

The parameters controlling this behavior are prefixed by `rdcc`, for _raw data chunk cache_. They apply to all datasets unless specifically changed for each one.

*   `rdcc_nbytes` sets the total size (measured in bytes) of the raw data chunk cache for each dataset. The default size is 1 MiB. This should be set to the size of each chunk times the number of chunks that are likely to be needed in cache.

*   `rdcc_w0` sets the policy for chunks to be removed from the cache when more space is needed. If the value is set to 0, then the library will always evict the least recently used chunk in cache. If the value is set to 1, the library will always evict the least recently used chunk which has been fully read or written, and if none have been fully read or written, it will evict the least recently used chunk. If the value is between 0 and 1, the behavior will be a blend of the two. Therefore, if the application will access the same data more than once, the value should be set closer to 0, and if the application does not, the value should be set closer to 1.

*   `rdcc_nslots` is the number of chunk slots in the cache for each dataset. In order to allow the chunks to be looked up quickly in cache, each chunk is assigned a unique hash value that is used to look up the chunk. The cache contains a simple array of pointers to chunks, which is called a hash table. A chunk’s hash value is simply the index into the hash table of the pointer to that chunk. While the pointer at this location might instead point to a different chunk or to nothing at all, no other locations in the hash table can contain a pointer to the chunk in question. Therefore, the library only has to check this one location in the hash table to tell if a chunk is in cache or not. This also means that if two or more chunks share the same hash value, then only one of those chunks can be in the cache at the same time. When a chunk is brought into cache and another chunk with the same hash value is already in cache, the second chunk must be evicted first. Therefore it is very important to make sure that the size of the hash table (which is determined by the `rdcc_nslots` parameter) is large enough to minimize the number of hash value collisions. Due to the hashing strategy, this value should ideally be a prime number. As a rule of thumb, this value should be at least 10 times the number of chunks that can fit in `rdcc_nbytes` bytes. For maximum performance, this value should be set approximately 100 times that number of chunks. The default value is 521.

Chunks and caching are described in greater detail in the HDF5 documentation.

Data alignment
---------------

When creating datasets within files, it may be advantageous to align the offset within the file itself. This can help optimize read and write times if the data become aligned with the underlying hardware, or may help with parallelism with MPI. Unfortunately, aligning small variables to large blocks can leave a lot of empty space in a file. To this effect, application developers are left with two options to tune the alignment of data within their file. The two variables `alignment_threshold` and `alignment_interval` in the `File` constructor help control the threshold in bytes where the data alignment policy takes effect and the alignment in bytes within the file. The alignment is measured from the end of the user block.

For more information, see the official HDF5 documentation H5P\_SET\_ALIGNMENT.

Meta block size
----------------

Space for metadata is allocated in blocks within the HDF5 file. The argument `meta_block_size` of the `File` constructor sets the minimum size of these blocks. Setting a large value can consolidate metadata into a small number of regions. Setting a small value can reduce the overall file size, especially in combination with the `libver` option. This controls how the overall data and metadata are laid out within the file.

For more information, see the official HDF5 documentation H5P\_SET\_META\_BLOCK\_SIZE.

Reference
----------

Note

Unlike Python file objects, the attribute `File.name` gives the HDF5 name of the root group, “`/`”. To access the on-disk name, use `File.filename`.

_class_ h5py.File(_name_, _mode\='r'_, _driver\=None_, _libver\=None_, _userblock\_size\=None_, _swmr\=False_, _rdcc\_nslots\=None_, _rdcc\_nbytes\=None_, _rdcc\_w0\=None_, _track\_order\=None_, _fs\_strategy\=None_, _fs\_persist\=False_, _fs\_threshold\=1_, _fs\_page\_size\=None_, _page\_buf\_size\=None_, _min\_meta\_keep\=0_, _min\_raw\_keep\=0_, _locking\=None_, _alignment\_threshold\=1_, _alignment\_interval\=1_, _\*\*kwds_)

Open or create a new file.

Note that in addition to the `File`\-specific methods and properties listed below, `File` objects inherit the full interface of `Group`.

Parameters:

*   **name** – Name of file (os.PathLike\[str\], bytes or str), or an instance of `h5f.FileID` to bind to an existing file identifier, or a file-like object (see Python file-like objects).

*   **mode** – Mode in which to open file; one of (“w”, “r”, “r+”, “a”, “w-“). See Opening & creating files.

*   **driver** – File driver to use; see File drivers.

*   **libver** – Compatibility bounds; see Version bounding.

*   **userblock\_size** – Size (in bytes) of the user block. If nonzero, must be a power of 2 and at least 512. See User block.

*   **swmr** – If `True` open the file in single-writer-multiple-reader mode. Only used when mode=”r”.

*   **rdcc\_nbytes** – Total size of the raw data chunk cache in bytes. The default size is \\(1024^2\\) (1 MiB) per dataset.

*   **rdcc\_w0** – Chunk preemption policy for all datasets. Default value is 0.75.

*   **rdcc\_nslots** – Number of chunk slots in the raw data chunk cache for this file. Default value is 521.

*   **track\_order** – Track dataset/group/attribute creation order under root group if `True`. Default is `h5.get_config().track_order`.

*   **fs\_strategy** – The file space handling strategy to be used. Only allowed when creating a new file. One of “fsm”, “page”, “aggregate”, “none”, or `None` (to use the HDF5 default).

*   **fs\_persist** – A boolean to indicate whether free space should be persistent or not. Only allowed when creating a new file. The default is False.

*   **fs\_page\_size** – File space page size in bytes. Only use when fs\_strategy=”page”. If `None` use the HDF5 default (4096 bytes).

*   **fs\_threshold** – The smallest free-space section size that the free space manager will track. Only allowed when creating a new file. The default is 1.

*   **page\_buf\_size** – Page buffer size in bytes. Only allowed for HDF5 files created with fs\_strategy=”page”. Must be a power of two value and greater or equal than the file space page size when creating the file. It is not used by default.

*   **min\_meta\_keep** – Minimum percentage of metadata to keep in the page buffer before allowing pages containing metadata to be evicted. Applicable only if `page_buf_size` is set. Default value is zero.

*   **min\_raw\_keep** – Minimum percentage of raw data to keep in the page buffer before allowing pages containing raw data to be evicted. Applicable only if `page_buf_size` is set. Default value is zero.

*   **locking** –

The file locking behavior. One of:

*   False (or “false”) – Disable file locking

*   True (or “true”) – Enable file locking

*   ”best-effort” – Enable file locking but ignore some errors

*   None – Use HDF5 defaults

Warning

The HDF5\_USE\_FILE\_LOCKING environment variable can override this parameter.

Only available with HDF5 >= 1.12.1 or 1.10.x >= 1.10.7.

*   **alignment\_threshold** – Together with `alignment_interval`, this property ensures that any file object greater than or equal in size to the alignment threshold (in bytes) will be aligned on an address which is a multiple of alignment interval.

*   **alignment\_interval** – This property should be used in conjunction with `alignment_threshold`. See the description above. For more details, see Data alignment.

*   **meta\_block\_size** – Determines the current minimum size, in bytes, of new metadata block allocations. See Meta block size.

*   **kwds** – Driver-specific keywords; see File drivers.

_classmethod_ in\_memory(_file\_image\=None_, _block\_size\=64 \* 1024_, _\*\*kwargs_)

Parameters:

*   **file\_image** – The initial file contents as bytes (or anything that supports the Python buffer interface). HDF5 takes a copy of this data.

*   **block\_size** – Chunk size for new memory alloactions (default 64 KiB).

Other keyword arguments are like `File`, although name, mode, driver and locking can’t be passed.

\_\_bool\_\_()

Check that the file descriptor is valid and the file open:

\>>> f \= h5py.File(filename)
\>>> f.close()
\>>> if f:
...     print("file is open")
... else:
...     print("file is closed")
file is closed

close()

Close this file. All open objects will become invalid.

flush()

Request that the HDF5 library flush its buffers to disk.

id

Low-level identifier (an instance of `FileID`).

filename

Name of this file on disk, as a Unicode string.

mode

String indicating if the file is open readonly (“r”) or read-write (“r+”). Will always be one of these two values, regardless of the mode used to open the file.

swmr\_mode

True if the file access is using Single Writer Multiple Reader (SWMR). Use `mode` to distinguish SWMR read from write.

driver

String giving the driver used to open the file. Refer to File drivers for a list of drivers.

libver

2-tuple with library version settings. See Version bounding.

userblock\_size

Size of user block (in bytes). Generally 0. See User block.

meta\_block\_size

Minimum size, in bytes, of metadata block allocations. Default: 2048. See Meta block size.
