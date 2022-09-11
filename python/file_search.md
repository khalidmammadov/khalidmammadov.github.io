# Python file search for "unlucky" ones
<!-- wp:paragraph -->
<p>If you happened to have to work with a some OS that lacks proper file search tool then you might find this article and code useful. I have seen so many users and developers working on Windows OS to use third party tools to search for a file or some other data within files so many times that I have decided to put together a code that will show how it's easy to make a tool to search for a file or text on any operating system. I have tested it on POSIX and Windows platforms.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Please find the full source code on below address:</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://github.com/khalidmammadov/python_code/tree/master/file_search">https://github.com/khalidmammadov/python_code/tree/master/file_search</a></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Usage</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>You can execute it from your command line by simple running:</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>(<em>Note: apologies for below samples as they all are coming from my Linux machine and not Windows but you can run them on Windows with no issues)</em></p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">python file_search/find_file.py --dir /home/khalid/dev --filename find.py</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>Or if you want to look for a text within file then use below:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">python file_search/find_file.py --dir /home/khalid/dev --filename *.py --text hello</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>and you should receive output similar to below</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">**********************
Searching text:    hello
Inside dir:        /home/khalid/dev
With file pattern: *.py
**********************
RESULT:
Found in file: /home/khalid/dev/python_doc_changes/lib/python3.8/site-packages/pyparsing.py
        Line no:"2982" ***:     hello = "Hello, World!"
        Line no:"2983" ***:     print (hello, "-&gt;", greet.parseString(hello))
        Line no:"5065" ***:             hello = "Hello, World!"
        Line no:"5066" ***:             print (hello, "-&gt;", greet.parseString(hello))
Found in file: /home/khalid/dev/python_doc_changes/lib/python3.8/site-packages/pip/utils/__init__.py
        Line no:"1769" ***:            print('hello')
        Line no:"1770" ***:        self.assertEqual(stdout.getvalue(), 'hello\n')
Found in file: /home/khalid/dev/python_doc_changes/lib/python3.8/site-packages/babel/messages/pofile.py
        Line no:"3211" ***:     &gt;&gt;&gt; print(unescape('"Say:\\n  \\"hello, world!\\"\\n"'))
        Line no:"3213" ***:       "hello, world!"
        Line no:"3236" ***:     ... "  \"hello, world!\"\n"'''))
        Line no:"3238" ***:       "hello, world!"
        Line no:"3579" ***:     ...   "hello, world!"
        Line no:"3581" ***:     '"Say:\\n  \\"hello, world!\\"\\n"'</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading -->
<h2>The code explained</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This section is for curious Python developers. I will list the main functions and modules I have used in this projects to make this code as simple as possible.</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>os.walk()</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>I have used <strong>walk</strong> function from <strong>os</strong> module to traverse a folder structure and look for all files within. It returns a tuple with a folder name and a list with file names within that folder e.g.:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">for w in os.walk('/usr/lib/'):
    print(w)
&gt;&gt;&gt; ('/usr/lib/firefox/distribution/searchplugins/locale/en-GB', [], ['amazon-en-GB.xml', 'google.xml', 'ddg.xml'])
&gt;&gt;&gt; ('/usr/lib/firefox/distribution/searchplugins/locale/en-US', [], ['amazondotcom.xml', 'google.xml', 'ddg.xml'])
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>fnmatch.filter()</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>I am using this function few times in the code to filter files that match the pattern or a folder name that match e.g.:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">fnmatch.filter(["ab.py", "v.py", "c.py"], "?.py"))
&gt;&gt;&gt; ['v.py', 'c.py']</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>fileinput.input()</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This is very convenient function from a standard library to loop over list of files and open them them one by one process as needed. It also provided convenient methods to check which file is being processed and line number. Using this function I am checking for a text within a file and printing the line where it found along side with line number</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Summary</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This code demonstrates how it's straightforward to build a search tool in Python and use it on any platform. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>I hope you found this article useful and interesting please feel free to comment and commit should you wish to.</p>
<!-- /wp:paragraph -->