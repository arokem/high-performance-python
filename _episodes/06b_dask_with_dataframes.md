---
title: "Dask Dataframes"
teaching: 5
exercises: 0
questions:
- "How can we replace some Pandas code with Dask?"
objectives:
- "Use Dask DataFrames to analyze data from CSV files"
keypoints:
- "Dask DataFrames can speed up things even more, for tabular data"
---

For example, let's say that we'd like to analyze a large data that resides in a
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

