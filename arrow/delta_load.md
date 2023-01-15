# Loading Delta (Parquet) files into Apache Arrow

Here I would like to show how to read Delta table into Apache Arrow in-memory table format.

I would like to mention that although Delta format is very similar to Parquet with regards how to how actual data is stored 
they differ in way how we read this data. Delta has got additional versioning and metadata information that
dictates how underlying Parquet data must be accessed.

In this respect, I would like to first show how we can load a Parquet files inside Delta table into the Arrow, 
then load as Delta. Loading Delta as Parquet would yield incorrect results as I will demonstrate shortly.

_You can download source code here: [Delta to Arrow Jupiter Notebook source](https://github.com/khalidmammadov/python_code/blob/master/arrow_delta/Delta_to_Arrow.ipynb)_

### Environment set up

_You can skip this part if you don't need setup._ 

For this example I am using Docker and Jupiter Notebook. There is a very convenient Docker image provided by Jupiter
project that contain not only Jupiter Notebook bu also  all necessary tools for Data Science projects 
(e.g. pandas, numpy, scipy, pyspark etc.).

It's very easy to start once you have Docker on your VM:

```shell
khalid@vm:~$ docker run -it --rm -p 10000:8888 -v "${PWD}":/home/jovyan/work jupyter/all-spark-notebook:latest
```
Explanation fo arguments:
- It maps my current directory to a `work` directory on a container so if I want to save my notebook 
then I can just save that into `work` folder.
- It also maps local 8888 port to external 10000 port so we can access it via our browser. 
- Once container stopped it will be removed

It will print URL with a token you can copy and paste in your browser to start interacting with Jupiter Notebook.

```shell
Executing the command: jupyter lab
[I 2023-01-15 10:12:55.551 ServerApp] jupyter_server_terminals | extension was successfully linked.
[I 2023-01-15 10:12:55.555 ServerApp] jupyterlab | extension was successfully linked.
[W 2023-01-15 10:12:55.557 NotebookApp] 'ip' has moved from NotebookApp to ServerApp. This config will be passed to ServerApp. Be sure to update your config before our next release.
[W 2023-01-15 10:12:55.557 NotebookApp] 'port' has moved from NotebookApp to ServerApp. This config will be passed to ServerApp. Be sure to update your config before our next release.
[W 2023-01-15 10:12:55.557 NotebookApp] 'port' has moved from NotebookApp to ServerApp. This config will be passed to ServerApp. Be sure to update your config before our next release.
[I 2023-01-15 10:12:55.560 ServerApp] nbclassic | extension was successfully linked.
[I 2023-01-15 10:12:55.560 ServerApp] Writing Jupyter server cookie secret to /home/jovyan/.local/share/jupyter/runtime/jupyter_cookie_secret
[I 2023-01-15 10:12:55.812 ServerApp] notebook_shim | extension was successfully linked.
[I 2023-01-15 10:12:56.065 ServerApp] notebook_shim | extension was successfully loaded.
[I 2023-01-15 10:12:56.066 ServerApp] jupyter_server_terminals | extension was successfully loaded.
[I 2023-01-15 10:12:56.066 LabApp] JupyterLab extension loaded from /opt/conda/lib/python3.10/site-packages/jupyterlab
[I 2023-01-15 10:12:56.066 LabApp] JupyterLab application directory is /opt/conda/share/jupyter/lab
[I 2023-01-15 10:12:56.069 ServerApp] jupyterlab | extension was successfully loaded.
[I 2023-01-15 10:12:56.073 ServerApp] nbclassic | extension was successfully loaded.
[I 2023-01-15 10:12:56.074 ServerApp] Serving notebooks from local directory: /home/jovyan
[I 2023-01-15 10:12:56.074 ServerApp] Jupyter Server 2.0.6 is running at:
[I 2023-01-15 10:12:56.074 ServerApp] http://ea15a5075259:8888/lab?token=<<TOKEN>>
[I 2023-01-15 10:12:56.074 ServerApp]  or http://127.0.0.1:8888/lab?token=<<TOKEN>>
[I 2023-01-15 10:12:56.074 ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 2023-01-15 10:12:56.077 ServerApp] 
    
    To access the server, open this file in a browser:
        file:///home/jovyan/.local/share/jupyter/runtime/jpserver-7-open.html
    Or copy and paste one of these URLs:
        http://ea15a5075259:8888/lab?token=<<TOKEN>>
     or http://127.0.0.1:8888/lab?token=<<TOKEN>>
```

### Sample data  

Let's start with creating a sample Pandas DataFrame and save that into a Delta file

```python
import pandas as pd
from deltalake.writer import write_deltalake
df = pd.DataFrame({"id": list(range(1,5))})
```

```python
df.head
```

```text
    <bound method NDFrame.head of    id
    0   1
    1   2
    2   3
    3   4>
```
```python
write_deltalake("sample-delta-table",df)
```

### Read as Parquet into Arrow dataset

Now, lets read that table as Parquet and see how it looks:

```python
import pyarrow.parquet as pq
import pyarrow.compute as pc
```


```python
delta_ds = ds.dataset("sample-delta-table", format="parquet")
```

```python
delta_ds.files
```

    ['sample-delta-table/0-f2e2ec6d-f585-4175-b5d2-36857d3f6fc1-0.parquet']

As you can see the table consist of only one source file. 


And printing actual data shows all as expected, and we have got the same data as in our original Pandas DataFrame  
```python
for batch in delta_ds.to_batches():
    print(batch["id"])
```

    [
      1,
      2,
      3,
      4
    ]

### Overwrite saved Delta table data and read again

Now, lets see what is going to happen if we overwrite Delta table data and read that data as Parquet again.

Redefine a DataFrame with new values:
```python
df = pd.DataFrame({"id": list(range(5,10))})
```
Write into files:
```python
write_deltalake("sample-delta-table", df, mode='overwrite')
```

And read into Arrow as Parquet and show files it consist of:
```python
delta_ds = ds.dataset("sample-delta-table", format="parquet")
delta_ds.files
```

As you can see we have got original file and a new one:

    ['sample-delta-table/0-f2e2ec6d-f585-4175-b5d2-36857d3f6fc1-0.parquet',
     'sample-delta-table/1-40aa8bdd-b45c-4b7d-8bda-fc8b417f603a-0.parquet']


Now, if we print actual data from the table:
```python
for batch in delta_ds.to_batches():
    print(batch["id"])
```

We can now see that it shows original data a new one instead of only new one as we did execute overwrite.

    [
      1,
      2,
      3,
      4
    ]
    [
      5,
      6,
      7,
      8,
      9
    ]

So, as I said at beginning Delta uses additional metadata to read only actual data and reading that as Parquet is incorrect.  

### Read as Delta

Now, lets load that as Delta respecting to metadata. For this we will need to import `deltalake` package which
is using Rust implementation of the Delta library and has Python bindings to be able to use with Python.

```python
!pip install deltalake
```

    Requirement already satisfied: deltalake in /opt/conda/lib/python3.10/site-packages (0.6.4)
    Requirement already satisfied: pyarrow>=7 in /opt/conda/lib/python3.10/site-packages (from deltalake) (10.0.1)
    Requirement already satisfied: numpy>=1.16.6 in /opt/conda/lib/python3.10/site-packages (from pyarrow>=7->deltalake) (1.23.5)

Import the lib and read the table and show version number. Which is how Delta versions the changes to the data:
```python
from deltalake import DeltaTable

dt = DeltaTable("sample-delta-table")
dt.version()
```

    1

So, the version is 1, and it starts from 0 and shows that there were 2 changes and we can load and look to the table as v1 or as it was on v0 ! 

Delta table conveniently provide a function to read data into Arrow format: 
```python
arrow_ds = dt.to_pyarrow_table()
```

Now, lets check the data:
```python
for batch in arrow_ds.to_batches():
    print(batch["id"])
```

    [
      5,
      6,
      7,
      8,
      9
    ]

As you can see it only shows latest and current version of data from Delta table.

