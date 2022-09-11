# Deterministic in Python (lru_cache) for function optimisation
<!-- wp:paragraph -->
<p>If you ever worked in Oracle you might know that a function within DB can be marked as deterministic. Which means that if you call a function few times with the same arguments it must save the result on the first run so it can return it next time without re-executing the body of the function. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Of course, that also means that a function must also not have any side effects and not be dependent from an external data that is prone to change.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>Use case</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>In python if you are processing an immutable data type and the function body does not have any side effect then you can benefit from it.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>Benefits</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Benefits using deterministic function is that you can gain tremendous performance gains as function does not have to compute/execute the function body when it sees that it has encountered similar call in the past.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>In Python</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Python provide function decorator called <strong>lru_cache</strong> from <strong>functools</strong> module. This wraps the original function and provide convenient way to achieve a deterministic feature.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>Example</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p><a href="https://github.com/khalidmammadov/python_code/tree/master/deterministic">https://github.com/khalidmammadov/python_code/tree/master/deterministic</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Below example count words for a given sentence and remembers the result. The loop feeds that sentence 3 times and each time we measure how much time it took to execute it:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">import collections
import functools
import time
@functools.lru_cache(maxsize=200)
def count_words(sentence):
    _count = collections.Counter()
    for w in str(sentence).split():
        _count[w] += 1
    return _count
if __name__ == "__main__":
    sen = """In mathematics, computer science and physics, 
            a deterministic system is a system in which no 
            randomness is involved in the development of future states of the system"""
    times = []
    for i in range(3):
        start_time = time.perf_counter()
        count = count_words(sen)
        stop_time = time.perf_counter()
        time_taken = stop_time-start_time
        times.append(time_taken)
        print("Exec #{} time: {:.9f} performance gain {:%}".format(i, time_taken, 1-times[i]/times[0]))
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>And if you run it then you should get output like this:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">khalid@ubuntu:~/dev/python_code$ python deterministic/determtic.py 
Exec #0 time: 0.000018115 performance gain 0.000000%
Exec #1 time: 0.000000628 performance gain 96.533256%
Exec #2 time: 0.000000310 performance gain 98.288714%
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>As you can see there is whooping <strong>98.3%</strong> improvement in performance.</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>But please be aware that above code uses cache limit of 200 unique argument calls to reduce caching for rarely used sentences for example. But the decorator supports omitting this parameter and essentially making it unlimited but that might increase memory consumption. So, be very careful not to use it excessively when it might affect whole system's performance. </p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Conclusion</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>In my opinion this is very powerful Python feature and that every Python developer should keep handy to use in production code.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Hope you learned something today and enjoyed this article.  </p>
<!-- /wp:paragraph -->