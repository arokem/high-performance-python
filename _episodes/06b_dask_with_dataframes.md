---
title: "Dask Dataframes"
teaching: 5
exercises: 0
questions:
- "How can we replace some Pandas code with Dask?"
objectives:
- "Use Dask DataFrames to analyze data from CSV files"
keypoints:
- "Dask arrays can speed up things even more, for array data"
- "Dask provides a whole universe of tools for parallelization"
---


Dask has implementations of several commonly-used pythonic data-structures. In
particular, it implements a data structure that resembles the Numpy Array object
and another data structure that resembles the Pandas DataFrame. This lets us do
slightly more interesting things and leverage our knowledge of these tools. For
example, let's say that we'd like to analyze a large data that resides in a
collection of CSV files.


~~~
from glob import glob

fnames = glob('data/gapminder_gdp*.csv')
~~~
{: .python}


We use the Dask API to read in the data as a dask DataFrame:

~~~
import dask.dataframe as dd

accounts = dd.read_csv(filename)
~~~
{: .python}

This object implements many of the methods that the original Pandas DataFrame
implements. For example, try running:

~~~
accounts.columns
~~~
{: .python}

You'll see that

