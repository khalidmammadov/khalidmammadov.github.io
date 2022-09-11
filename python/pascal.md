# Pascal's Triangle in Python

<!-- wp:heading {"level":3} -->
<h3>Overview</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>In this article I am putting together a code that calculates Pascal's Triangle in Python code. I am going to use Python 3.8's new <strong>Combinatorics</strong> function that was added into standard library under the <strong>math</strong> module.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://en.wikipedia.org/wiki/Pascal%27s_triangle">Pascal's triangle</a> consists from triangular array of the binomial coefficients and looks as below:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">                 1
                1 1
               1 2 1
              1 3 3 1
             1 4 6 4 1
           1 5 10 10 5 1
          1 6 15 20 15 6 1
        1 7 21 35 35 21 7 1
       1 8 28 56 70 56 28 8 1
    1 9 36 84 126 126 84 36 9 1</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>So every number is a sum of top two number just above that number. But to get every number <a href="https://en.wikipedia.org/wiki/Binomial_coefficient">binomial coefficient</a> must computed and it's done by calculating number of Combinations for specific place by using <a href="https://en.wikipedia.org/wiki/Combination">combination formula</a> giving row number as number of elements and column no as number of choices.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>I have already compiled the code and it can be accessed in below address:</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://github.com/khalidmammadov/python_code/tree/master/pascal_triangle">https://github.com/khalidmammadov/python_code/tree/master/pascal_triangle</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>The main function that calculates triangle takes line number as an argument and builds a list of list where every list within top list corresponds to the line of the triangle i.e.</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">[['1'],
['1', '1'],
['1', '2', '1'],
['1', '3', '3', '1'],
['1', '4', '6', '4', '1'],
['1', '5', '10', '10', '5', '1'],
['1', '6', '15', '20', '15', '6', '1'],
['1', '7', '21', '35', '35', '21', '7', '1'],
['1', '8', '28', '56', '70', '56', '28', '8', '1'],
['1', '9', '36', '84', '126', '126', '84', '36', '9', '1']]
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>and this is done by below functions:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">def bin_expans(row, col):
    """
        Calculate binomial coefficient
    """
    nCr = math.comb(row, col)
    return nCr


def build_triangle(rows = 10):
    """
        Calculate triangle
    """
    triangle = [[str(bin_expans(row, col))
                    for col in range(row+1)]
                        for row in range(rows)]
    return triangle</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>This new function from math module makes the calculation easy and without using additional packages.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>The full source code is available at the URL mentioned above. Additionally the code defines pretty printing function and does version check and implement unit test for the calculations.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>Execution</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>If you clone the repository and ensure you have  python 3.8 installed in /usr/bin/python3.8 directory (for POSIX) or change the Makefile accordingly, you can execute the code by running below command:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">make unittest
make run</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>Conclusion</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This simple code demonstrate powerful and ever-expanding features of Python that we can use and contribute to. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>I hope you enjoyed reading or checking it out and wish you every success in Python journey! </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>And yes please do comment if you wish so.  </p>
<!-- /wp:paragraph -->