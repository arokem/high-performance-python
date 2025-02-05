---
title: "Dask arrays"
teaching: 10
exercises: 0
questions:
- "Can we speed up array computations?"
objectives:
- "Use Dask arrays to analyze data that naturally comes in arrays"
keypoints:
- "Dask arrays can speed up things even more, for array data"
---



Let's do something slightly more interesting (and neuro-related):

Let's say that we'd like to compute the tsnr over several runs of
fMRI data, for example, using the open-fMRI dataset ds000114:

~~~
from glob import glob

fnames = glob('/data/ds000114/sub-01/ses-test/func/*.nii.gz')
~~~
{: .python}

This is a list with 5 different file-names, for the different runs during
this session.

One way to calculate the tsnr across is to loop over the
files, read in the data for each one of them, concatenate the data and then
compute the tsnr from the concatenated series:

~~~
data = []
for fname in fnames:
    data.append(nib.load(fname).get_data())

data = np.concatenate(data, -1)
tsnr = data.mean(-1) / data.std(-1)
~~~
{: .python}

> #### Lazy loading in Nibabel
> Nibabel uses "lazy loading". That means that data are not read from file
> until when the nibabel `load` function is called on a file-name. Instead,
> Nibabel waits until we ask for the data, using the `get_data` method of
> The `Nifti1Image` class to read the data from file.

When we do that, most of the time is spent on reading the data from file.
As you can probably reason yourself, the individual items in the data
list have no dependency on each other, so they could be calculated in
parallel.

Because of nibabel's lazy-loading, we can instruct it to wait with the
call to `get_data`. We create a delayed function that we call
`delayed_get_data`:

~~~
delayed_get_data = delayed(nib.Nifti1Image.get_data)
~~~
{: .python}

Then, we use this function to create a list of items and delay each one
of the computations on this list:

~~~
data = []
for fname in fnames:
    data.append(delayed_get_data(nib.load(fname)))

data = delayed(np.concatenate)(data, -1)
tsnr = delayed(data.mean)(-1) / delayed(data.std)(-1)
~~~
{: .python}

Dask figures that out for you as well:

~~~
tsnr.visualize()
~~~
{: .python}

![](../fig/dask_delayed_tsnrs.png)

And indeed computing tsnr this way can give you an approximately 4-fold
speedup. This is because Dask allows the Python process to read several
of the files in parallel, and that is the performance bottle-neck here.

### Dask arrays

This is already quite useful, but wouldn't you rather just tell dask that
you are going to create some data and to treat it all as delayed until
you are ready to compute the tsnr?

This idea is implemented in the dask array interface. The idea here is that
you create something that provides all of the interfaces of a numpy array, but
all the computations are treated as delayed.

This is what it would look like for the tsnr example. Instead of
appending delayed calls to `get_data` into the array, we create a series
of dask arrays, with `delayed_get_data`. We do need to know both the shape
and data type of the arrays that will eventually be read, but

~~~
import dask.array as da

delayed_arrays = []
for fname in fnames:
    img = nib.load(fname)
    delayed_arrays.append(da.from_delayed(delayed_get_data(img),
                          img.shape,
                          img.get_data_dtype()))
~~~
{: .python}


If we examine these variables, we'll see something like this:

~~~
[dask.array<from-value, shape=(64, 64, 30, 184), dtype=int16, chunksize=(64, 64, 30, 184)>,
 dask.array<from-value, shape=(64, 64, 30, 238), dtype=int16, chunksize=(64, 64, 30, 238)>,
 dask.array<from-value, shape=(64, 64, 30, 76), dtype=int16, chunksize=(64, 64, 30, 76)>,
 dask.array<from-value, shape=(64, 64, 30, 173), dtype=int16, chunksize=(64, 64, 30, 173)>,
 dask.array<from-value, shape=(64, 64, 30, 88), dtype=int16, chunksize=(64, 64, 30, 88)>]
~~~
{: .python}

These are notional arrays, that have not been materialized yet. The data
has not been read from memory yet, although dask already knows where it
would put them when they should be read.

We can use the `dask.array.concatenate` function:

~~~
arr = da.concatenate(delayed_arrays, -1)
~~~
{: .python}

And we can then use methods of the `dask.array` object to complete the
computation:

~~~
tsnr = arr.mean(-1) / arr.std(-1)
~~~
{: .python}

This looks exactly like the code we used for the numpy array!

Given more insight into what you want to do, dask is able to construct an
even more sophisticated task graph:

<img src="../fig/dask_array_tsnr.png" width="1200px"/>

This looks really complicated, but notice that because dask has even more
insight into what we are trying to do, it can delay some things until
after aggregation. For example, the square root computation of the
standard deviation can be done once at the end, instead of on each array
separately.

And this leads to an approximately additional 2-fold speedup.

One of the main things to notice about the dask array is that because the
data is not read into memory it can represent very large datasets, and
schedule operations over these large datasets in a manner that makes the code
seem as though all the data is in memory.