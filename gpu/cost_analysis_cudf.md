# Cloud Storage Ingestion Cost Estimation for Big Data using Monte Carlo Simulation over CUDA libraries and NVIDIA Tesla GPU

In this post I am going to use CUDA DataFrame API (cuDF) and Tesla GPU (P100) to
do statistical cost analysis using famous [Monte Carlo simulation](https://en.wikipedia.org/wiki/Monte_Carlo_method)

## Objective
In this example I am planning to estimate cost **range** with probabilities of a cloud ingestion,
based on historical data processing patters throughout the year. 

The plan is to use estimated daily data volume and simulate it over and over 
to get normal distribution (Monte Carlo) of the cost that can be used to estimate annual cost
of the ingestion that is often required by companies to do yearly cost planning.

Here assumption is that we ingest around 100 billion records throughout the day of various record sizes. 
And I also use a price per Gb for given Cloud provider to calculate total cost per day. 
In this case I picked up Azure Premium Storage Write cost which is £0.0231 per GB for UK South region.   

### Environment

Here I am using following distinctive toolsets (soft/hard):

- NVIDIA GPU (Tesla P100) 
- cuDF - RAPIDS API
- Jupiter notebook
- Ubuntu 20.4

#### Notebook

_You can download below notebook [here](https://github.com/khalidmammadov/python_code/blob/master/notebooks/CloudStorageCostAnalysis_RAPIDS.ipynb)._

Start the Jupiter as usual:
```
jupyter-lab --allow-root --ip='0.0.0.0' --NotebookApp.token='aaaa'
```

Import all required packages:
```python
import cudf
import numpy as np
import matplotlib.pyplot as plt
import asyncio
```

### Generate input data 

In order to generate data I am going to define function that takes 
mean and standard deviation and generates random data using Normal Distribution
with given sample size. 

```python
mu = 100 # Billion
days = 365
def gen_data(mean, std_dev, count):
    return np.abs(np.random.normal(mean, std_dev, count))
```

First generate sample records that calibrates around 100 billion
to simulate different record counts for different days.
```python
rows = np.round(gen_data(mu, 50, days), 0) * 1e9
rows[:50]
```
```text
array([1.34e+11, 8.00e+10, 8.10e+10, 1.61e+11, 5.30e+10, 1.00e+10,
       8.00e+09, 1.63e+11, 4.90e+10, 1.56e+11, 1.44e+11, 1.32e+11,
       1.45e+11, 5.20e+10, 1.23e+11, 1.06e+11, 2.40e+10, 6.50e+10,
       8.70e+10, 1.14e+11, 1.00e+11, 6.60e+10, 2.02e+11, 1.70e+11,
       1.23e+11, 3.70e+10, 9.70e+10, 1.04e+11, 4.40e+10, 1.10e+11,
       2.60e+10, 5.20e+10, 1.62e+11, 8.60e+10, 1.12e+11, 5.90e+10,
       1.01e+11, 4.60e+10, 4.00e+10, 8.40e+10, 3.70e+10, 1.32e+11,
       6.80e+10, 8.60e+10, 1.42e+11, 1.29e+11, 5.60e+10, 2.56e+11,
       1.33e+11, 1.77e+11])
```

Then generate different record sizes and again taking into account element of randomness 
```python
size_kb = gen_data(1, 0.5, days)
size_kb[:50]
```
```text
array([0.58665729, 1.45003301, 0.95157851, 1.21846504, 0.73394111,
       0.04577056, 1.09688516, 1.33025666, 0.09185652, 1.33208538,
       1.88424182, 1.1644085 , 0.97867671, 1.0765287 , 0.50662871,
       0.08066607, 1.1617845 , 1.08134547, 1.21386616, 1.22388331,
       0.58146822, 0.95853788, 0.92052233, 1.27611515, 0.55276376,
       0.84961316, 0.93015397, 0.27660171, 0.78476622, 0.19023719,
       1.59099942, 0.42745015, 1.04264242, 0.79059961, 0.9590777 ,
       1.2093227 , 0.67432358, 0.24007244, 0.21427387, 1.69870793,
       1.28338128, 0.9678571 , 1.13489447, 1.00205534, 0.93852447,
       0.4696987 , 1.10678811, 2.39465893, 1.42679065, 0.9891491 ])
```

Now, lets create CUDA DataFrame (which is very similar to Pandas one)
```python
cdf = cudf.DataFrame({"rows": rows, "size": size_kb})
type(cdf)
```
```text
cudf.core.dataframe.DataFrame
```

Then lets compute total size in Gb and total cost using unit price from a cloud provider 
I mentioned at the beginning:
```python
cdf["total_size_gb"] = np.round((cdf["rows"] * cdf["size"])/1024/1024)
cdf["cost"] = cdf["total_size_gb"] * 0.0231
```

Lets see how data now looks like. Please note that data is located inside GPU memory and not in the RAM.
```text
	rows 	size 	total_size_gb 	cost
0 	1.340000e+11 	0.586657 	74970.0 	1731.8070
1 	8.000000e+10 	1.450033 	110629.0 	2555.5299
2 	8.100000e+10 	0.951579 	73507.0 	1698.0117
3 	1.610000e+11 	1.218465 	187085.0 	4321.6635
4 	5.300000e+10 	0.733941 	37097.0 	856.9407
... 	... 	... 	... 	...
360 	1.170000e+11 	0.627697 	70038.0 	1617.8778
361 	1.490000e+11 	1.553880 	220802.0 	5100.5262
362 	1.030000e+11 	0.926896 	91048.0 	2103.2088
363 	6.000000e+10 	0.446918 	25573.0 	590.7363
364 	6.900000e+10 	0.662074 	43567.0 	1006.3977

365 rows × 4 columns
```

Let's also plot total cost to see how it varies:
```text
fig = plt.figure()
ax = fig.add_subplot()
ax.plot(cdf["cost"])
```
![total_cost](../images/storage_cost_cudf.png)

## Monte Carlo simulation

In order to do Monte Carlo simulation I am going to use Python's asyncio library and run 
above instructions in parallel for 100000 years!

```python
mu = 100 # Billion
days = 365
def gen_data(mean, std_dev, count):
    return np.abs(np.random.normal(mean, std_dev, count))

# Monte Carlo
years = 100000
price_per_gb = 0.15214
async def generate_sample_sum():
    rows = np.round(gen_data(mu, 50, days), 0)*1e9
    size_kb = gen_data(1, 0.5, days)
    cdf = cudf.DataFrame({"rows": rows, "size": size_kb})
    cdf["total_size_gb"] = np.round((cdf["rows"] * cdf["size"])/(1024*2))
    cdf["cost"] = cdf["total_size_gb"] * price_per_gb
    return cdf["cost"].sum()
```

Run calculation in parallel:
```python
estimates = await asyncio.gather(*[generate_sample_sum() for _ in range(years)])
```
This will cause all calculations to be run in parallel on the GPU. As you can see it only uses 7% of the GPU
with only 530Mb (3%) to do calculations

![GPU_usage](../images/GPU_use_storage_cost_estimate.png)

## Result

Lets plot collected results and see outcomes:

```python
fig = plt.figure()
ax = fig.add_subplot()
ax.hist(estimates, 30, density=True)
fig.savefig("storage_cost_simulation_result")
```

Below is histogram of simulation results for 10000 years! Here x axis in pounds.
![simulation_result](../images/storage_cost_simulation_result.png)

Using this simulation one can calculate possible range of annual cost estimations with 
probabilities and that can be used to make cloud ingestion cost estimations for annual planning. 
