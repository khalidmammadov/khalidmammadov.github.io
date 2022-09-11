# Parallel processing (Pandas example)
<!-- wp:paragraph -->
<p>If you find yourself working with Pandas and need process loads of files in parallel then keep reading.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Here I explain how to perform parallel processing in Python in general and Pandas in particular example. As you know Python (CPython) uses single thread to run the program but Python provides <strong>multiprocessing</strong> module to help us run multiple threads. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Lets put together a simple code that we can run in single and parallel modes.</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Code</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>I have following test files in my local folder. These are some exchange rates from different periods. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://github.com/datasets/exchange-rates">https://github.com/datasets/exchange-rates</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>I am  intending to create a code that will process them and filter specific date.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Process single file function:</p>
<!-- /wp:paragraph -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">def process_file(f: Path, filter_string: str) -&gt; pd.DataFrame:
    <em>"""
    Process single file:
     Load as Pandas Dataframe and filter my matching filter_string
    """
    </em>df = pd.read_csv(f, header='infer')
    return df[df["Date"] == filter_string]</pre>
<!-- /wp:preformatted -->

<!-- wp:paragraph -->
<p>As you can see it takes file and loads that using Pandas and filters some data.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Sequential</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Below function loops exchange rates directory and process files in a sequence:</p>
<!-- /wp:paragraph -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">def process_files_sequentially(p: Path) -&gt; List:<br>    <em>"""<br></em><em>    Process files one at a time by looping the given directory<br></em><em>    """<br></em><em>    </em>all_dfs = []<br>    for f in p.glob('*.csv'):<br>        df = process_file(f, '1982-01-01')<br>        all_dfs.append(df)<br><br>    return all_dfs</pre>
<!-- /wp:preformatted -->

<!-- wp:heading {"level":3} -->
<h3>Parallel</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Now, in order to convert this to parallel processing I am going to use <a href="https://docs.python.org/3/library/multiprocessing.html#multiprocessing.pool.Pool"><strong>Pool</strong></a> class. This class is very easy to use and it provides methods called  <strong>map</strong> and <strong>starmap</strong> for parallelization. The main difference between two is that map only works on iterable of single objects as builtin map function but in parallel but starmap allows to us to use iterable with multiple arguments (list, tuple etc).</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Below is the code:</p>
<!-- /wp:paragraph -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">def process_files_parallel(p: Path) -&gt; List:<br>    <em>"""<br></em><em>    Process files in the folder by creating parallel loads<br></em><em>    It will create N parallel processes based on CPU count (vertical scaling)<br></em><em>    """<br></em><em>    </em>p_args = []<br>    for f in p.glob('*.csv'):<br>        p_args.append([f, '1982-01-01'])<br><br>    with mpp.Pool() as p:<br>        all_dfs = p.starmap(process_file, p_args)<br><br>    return all_dfs</pre>
<!-- /wp:preformatted -->

<!-- wp:paragraph -->
<p>So here we loop as in single processing version but this time we build a list of parameters list (List[List]).  Then we provide this list of arguments along with function to the startmap method. The way Pool works it creates a "pool" of threads that can be utilized. If we dont provide a number argument to the Pool class it will use number of CPUs we have got as an argument and will create that amount of threads. So, in my example I have got 4 CPU (core) and it will create the same number of threads for processing. Then it runs our process_file function for the first four files from the list in parallel with arguments and once done will move to the next ones (depending which one finished first).</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Below is the test scripts I have run on my local REPL using timeit module for <strong>1000</strong> times to demonstrate how sequential processing version is different from parallel.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>Sequential test</h4>
<!-- /wp:heading -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">import timeit 
timeit.timeit("pp.process_files_sequentially(Path('~/dev/test/exchange-rates/data/').expanduser())", "from parallel_pandas import parallel as pp; from pathlib import Path", number=1000)
</pre>
<!-- /wp:preformatted -->

<!-- wp:paragraph -->
<p>Below is the screenshot from my process monitor. As you can see it only uses one CPU (CPU1) on 100% utilization and all others are idling.</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":451,"sizeSlug":"large"} -->
<figure class="wp-block-image size-large"><img src="http://www.khalidmammadov.co.uk/wp-content/uploads/2020/09/Sequential_1CPU-1024x225.png" alt="" class="wp-image-451"/></figure>
<!-- /wp:image -->

<!-- wp:heading {"level":4} -->
<h4>Parallel test</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>In below example I use parallel version of the processing method and run it again 1000 times</p>
<!-- /wp:paragraph -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">import timeit 
timeit.timeit("pp.process_files_parallel(Path('~/dev/test/exchange-rates/data/').expanduser())", "from parallel_pandas import parallel as pp; from pathlib import Path", number=1000)</pre>
<!-- /wp:preformatted -->

<!-- wp:paragraph -->
<p>Here is the screenshot from process monitor while processing</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":447,"sizeSlug":"large"} -->
<figure class="wp-block-image size-large"><img src="http://www.khalidmammadov.co.uk/wp-content/uploads/2020/09/Parallel_CPU-1024x221.png" alt="" class="wp-image-447"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>As you can see this time I am making use of all CPU power from my PC and processing data in parallel. This method reduces processing time by factor of your CPU/Cores </p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Git</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Please find below the source code:</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://github.com/khalidmammadov/python_code/tree/master/parallel_pandas">https://github.com/khalidmammadov/python_code/tree/master/parallel_pandas</a></p>
<!-- /wp:paragraph -->