# Python, Pandas, SQLAlchemy, SQL Server and Docker
<!-- wp:paragraph -->
<p>In this article I am going to walk through a simple application that makes use of Pandas, SQLAlchemy and SQLServer on Docker. This combination is part of a project I have been working on recently that is using complex Cash flow analytical computation process that moves financial data from one stage to another, massaging, transforming, merging with various sources and finally calculating to generate final data to store in the SQL DB.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Here, I am going to explain steps that needed to performed to create an environment and will put together a small app that demonstrate how one can utilize these technologies together. The app will load data from a csv file into a <strong>Pandas</strong>' <strong>DataFrame</strong> and then save it into <strong>SQL Server</strong> DB using <strong>pyodbc</strong> and <strong>SQLAlchemy</strong>. </p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2>Set up</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Before we start lets create development environment by setting up SQL Server 2017 database on a Docker container were we are going to save final data for persistence or to be consumed by other applications like reporting tools. Then we need to set up ODBC drivers for Linux so we can connect to DB from our app. </p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>SQL Server on Docker</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Download docker image</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">docker pull microsoft/mssql-server-linux</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>Start a new container and specify DB sysadmin password</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">docker run -itd -e 'ACCEPT_EULA=Y' -e 'MSSQL_SA_PASSWORD=ChangePassw0rd!' \
   -p 1433:1433 --name sql \
    microsoft/mssql-server-linux:2017-latest
# Change password to a new one
sudo docker exec -it sql /opt/mssql-tools/bin/sqlcmd \
   -S localhost -U SA -P "ChangePassw0rd!" \
   -Q 'ALTER LOGIN SA WITH PASSWORD="NewPassw0rd!"'</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>Check image started</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">docker ps </pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>Create a new DB</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Attach to the image and connect to the DB</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">khalid@ubuntu:~/docker/sqlserver$ docker exec -it sql "bash"
root@9fe0f2c60294:/# /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "NewPassw0rd!"</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">Once you are in create test DB and a table</pre>
<!-- /wp:preformatted -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">1&gt; create database findb;
2&gt; go
1&gt; use findb;
2&gt; go
Changed database context to 'findb'.
1&gt; create table daily_rates (RateDate date, Country varchar(256), Value decimal(20,8));
2&gt; go
1&gt; select * from daily_rates;
2&gt; go
RateDate         Country                                                                                                                                                                                                                                                          Value                 
---------------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ----------------------
(0 rows affected)
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">We will load data later on using pandas and SQLAlchemy</pre>
<!-- /wp:preformatted -->

<!-- wp:heading {"level":4} -->
<h4>Install SQL ODBC drivers for Linux</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Following steps are copied from Microsoft's official <a href="https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver15">documentation</a>.  And choose instructions relevant to your distribution. I use <strong>Ubuntu Bionic Beaver</strong> (18.04), so second option in my case is relevant. But you can find instructions for other distributions as well on the above URL.</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">sudo su 
curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -
#Download appropriate package for the OS version
#Choose only ONE of the following, corresponding to your OS version
#Ubuntu 16.04
curl https://packages.microsoft.com/config/ubuntu/16.04/prod.list &gt; /etc/apt/sources.list.d/mssql-release.list
#Ubuntu 18.04
curl https://packages.microsoft.com/config/ubuntu/18.04/prod.list &gt; /etc/apt/sources.list.d/mssql-release.list
#Ubuntu 19.10
curl https://packages.microsoft.com/config/ubuntu/19.10/prod.list &gt; /etc/apt/sources.list.d/mssql-release.list
exit
sudo apt-get update
sudo ACCEPT_EULA=Y apt-get install msodbcsql17
# optional: for bcp and sqlcmd
sudo ACCEPT_EULA=Y apt-get install mssql-tools
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' &gt;&gt; ~/.bash_profile
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' &gt;&gt; ~/.bashrc
source ~/.bashrc
# optional: for unixODBC development headers
sudo apt-get install unixodbc-dev</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>Create ODBC settings file odbc.ini</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Get SQL server's Docker image's IP address by inspecting the docker bridge network (assuming you followed my instructions above and started it on the default bridge network) </p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code {"highlightLines":"11"} -->
<pre class="wp-block-syntaxhighlighter-code">khalid@ubuntu:~$ docker network inspect bridge
...
       "ConfigOnly": false,
        "Containers": {
            "9fe0f2c60294271e19fc3d27f59d56e3ecaf40afd5f5698452b3335a5a573678": {
                "Name": "sql1",
                "EndpointID": "e8afe3352d4a02b38340413ba559a8b8e388eea6412316b75f626bd21632791a",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>First create a temporary file like so</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">khalid@ubuntu:~$ echo "[MSSQLServerDatabase] 
Driver      = ODBC Driver 17 for SQL Server
Description = Connect to my SQL Server instance
Trace       = No
Server      = 172.17.0.2" &gt; odbc.tmp
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>Install ODBC DSN settings</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">khalid@ubuntu:~$ sudo odbcinst -i -s -f ~/odbc,tmp -l</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>Inspect if installation was successful. You should see the same lines as above in the below config file:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">khalid@ubuntu:~$ cat /etc/odbc.ini 
[MSSQLServerDatabase]
Driver=ODBC Driver 17 for SQL Server
Description=Connect to my SQL Server instance
Trace=No
Server=172.17.0.2
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>Create a new Python virtual environment</h4>
<!-- /wp:heading -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">khalid@ubuntu:~/dev/python_code/pd_sql$ python -m venv venv
# and activate
khalid@ubuntu:~/dev/python_code/pd_sql$ source venv/bin/activate
(venv) khalid@ubuntu:~/dev/python_code/pd_sql$</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>Install Pandas, SQLAlchemy and pyodbc</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>While you are in the activated virtual environment install required packages</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">(venv) khalid@ubuntu:~/dev/python_code/pd_sql$ pip install pandas sqlalchemy pyodbc</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p><strong>NOTE: While installing above you might encounter with following pyodbc compilation error. This is due to pyodbc need code compilation when installed onto POSIX (Linux in this case) systems and fails when can't find reuired C++ headers</strong>. </p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code"> -g -fwrapv -O2 -g -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2 -fPIC -DPYODBC_VERSION=4.0.28 -I/home/khalid/dev/python_code/pd_sql/venv/include -I/usr/include/python3.8 -c src/buffer.cpp -o build/temp.linux-x86_64-3.8/src/buffer.o -Wno-write-strings
    In file included from src/buffer.cpp:12:0:
    src/pyodbc.h:45:10: fatal error: Python.h: No such file or directory
     #include &lt;Python.h&gt;
              ^~~~~~~~~~
    compilation terminated.
    error: command 'x86_64-linux-gnu-gcc' failed with exit status 1
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p><strong>FIX: I have created a fix and filed a pull request to the owner and waiting for resolution.</strong> While it's getting solved you can pull the fixed package from my repo and install: <a href="https://github.com/khalidmammadov/pyodbc">https://github.com/khalidmammadov/pyodbc</a></p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">~/dev/python_code/pd_sql$ git clone https://github.com/khalidmammadov/pyodbc
(venv) khalid@ubuntu:~/dev/python_code/pd_sql/pyodbc$ git checkout compile_for_venv
Branch 'compile_for_venv' set up to track remote branch 'compile_for_venv' from 'origin'.
Switched to a new branch 'compile_for_venv'
(venv) khalid@ubuntu:~/dev/python_code/pd_sql/pyodbc$ cd ../
(venv) khalid@ubuntu:~/dev/python_code/pd_sql$ pip install pyodbc/
Processing ./pyodbc
Installing collected packages: pyodbc
  Running setup.py install for pyodbc ... done
Successfully installed pyodbc-4.0.29b2+compile.for.venv
(venv) khalid@ubuntu:~/dev/python_code/pd_sql$ 
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading -->
<h2>Pandas app</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Now we can start building an app that makes use of all above installed technologies. For testing purposes I am going to use test exchange rates data that can be downloaded from <a href="https://github.com/khalidmammadov/exchange-rates/blob/master/data/daily.csv">https://github.com/khalidmammadov/exchange-rates/blob/master/data/daily.csv</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Download or clone the code I have prepared as per below</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://github.com/khalidmammadov/python_code">https://github.com/khalidmammadov/python_code</a></p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">(venv) khalid@ubuntu:~/dev/$ git clone https://github.com/khalidmammadov/python_code
(venv) khalid@ubuntu:~/dev/$ cd python_code/pd_sql</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>Below is the main part of the app from __main__.py. Here App class is used to hold the state of the application in one place and enrich it with various app related shared attributes and methods. It takes various arguments from command line and main being password to the database if you are running it in Linux.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>CSV data is read into pandas DataFrame using delegated static method from App class. Then data type for date column is corrected due to underlying library bug (only on Linux for pyodbc package).  Then data is ready to be saved into the database</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>__main__.py</h4>
<!-- /wp:heading -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">    # Initialise a global app
    # this will hold the state and components of the app
    app = App(log_flag=logging_enabled,
              debug_flag=debug_enabled,
              debug_dir=debug_dir,
              dbpwd=dbpwd)
    # Main implementation
    csv_path = Path('~/docker/test/exchange-rates/data')
    daily_csv_file = 'daily.csv'
    # Read file to a DataFrame
    csv_df = app.get_csv_data(csv_path.joinpath(daily_csv_file))
    csv_df = csv_df.astype({'Date': 'datetime64'})
    csv_df = csv_df.rename(columns={'Date': 'RateDate'})
    csv_df = csv_df.fillna(0)
    # Check the data
    app.debug_df('Rates from CSV:', csv_df)
    # Save Rates Into DB
    app.save_rates_into_db(csv_df)</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>app.py</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>App define various logging and debug tasks first and then defines methods for the actual app tasks. </p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">class App(object):
    def __init__(self, log_flag=None, debug_flag=None, debug_dir=None, dbpwd=None):
        self.logger = App.set_up_logger(log_flag, debug_flag)
        self.debug_enabled = debug_flag
        self.debug_dir = None
        self.config = load_config('conf.ini')
        dsn_name = self.get_config('DSN', 'Name')
        self.db_connection = new_connection(dsn_name, dbpwd)
....
....
    @timing
    def get_config(self, section, name):
        return get_config(self.config, section, name)
    @staticmethod
    def get_csv_data(file):
        return pd.read_csv(file)
    @timing
    def save_rates_into_db(self, rates_df):
        # Get rates from DB
        rates_db_df = get_all_rates(self.db_connection)
        rates_db_df = rates_db_df.astype({'RateDate': 'datetime64'})
        # Left Outer Join so we check for existence
        joined = pd.merge(rates_df,
                          rates_db_df,
                          left_on=['RateDate', 'Country'],
                          right_on=['RateDate', 'Country'],
                          how='left',
                          suffixes=['', '_table'])
        # Separate Inserts and updates
        cond = joined['Value_table'].isnull()
        inserts = joined[cond]
        updates = joined[~cond]
        # Check
        self.debug_df('Inserts:', inserts)
        self.debug_df('Updates:', updates)
        if inserts.index.size &gt; 0:
            insert_rates(self.db_connection,
                         daily_rates_tbl,
                         inserts)
        if updates.index.size &gt; 0:
            update_rates(self.db_connection,
                         daily_rates_tbl,
                         updates)
    def __del__(self):
        if self.db_connection:
            self.db_connection.close()
            del self.db_connection</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading {"level":4} -->
<h4>def save_rates_into_db </h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This method reads from db first (for update purposes later in the code) and converts data into a DataFrame.  Then data from CSV and DB is joined to decide what need to be inserted and what for update. </p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Then inserts and updates are separated and ready to go into the DB.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Below functions from lib submodule are doing actual insertions and deletions using SQLAlchemy syntax.</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">def insert_rates(connection, table, inserts):
    # Make dictionary of parameters for the inserts
    ins_params = make_upsert_list(inserts)
    connection.execute(
        table.insert(),
        ins_params)
def update_rates(connection, table, updates):
    # Make dictionary of parameters for the updates
    upd_params = make_upsert_list(updates)
    # Execute statement (will autocommit)
    # Generate single updates
    for uld in upd_params:
        connection.execute(
            table.update().
                where(
                    and_(
                        table.c.RateDate == uld['RateDate'],
                        table.c.Country == uld['Country'])),
            upd_params)</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>In both cases the data first converted into a list of dicts so it can be parameterized for the DML operations.</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">def make_upsert_list(df):
    keys, data = df_to_sqla(df)
    return [dict(zip(keys, row)) for row in data]
</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p>Also the conversions:</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->
<pre class="wp-block-syntaxhighlighter-code">def convert_type(val):
    if isinstance(val, pd.Timestamp):
        val = val.to_pydatetime()
    elif isinstance(val, np.int64):
        val = int(val)
    elif isinstance(val, np.float64):
        val = float(val)
    return val
def df_to_sqla(df):
    keys = df.columns
    ncols = len(keys)
    nrows = len(df.index)
    print(df.dtypes)
    data_list = []
    for r in range(nrows):
        data_list.append([])
        for c in range(ncols):
            d = None
            val = df.iloc[r, c]
            val = convert_type(val)
            data_list[r].append(val)
    return keys, data_list</pre>
<!-- /wp:syntaxhighlighter/code -->

<!-- wp:heading -->
<h2>Conclusion</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This article demonstrates how one can spin up a database on a Docker container and then connect to it using Python (pyodbc, SQLAlchemy) and manipulate data as required using Pandas very easily.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Thanks for reading and hope you liked this article and learned something new today!</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":4} -->
<h4>References</h4>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Below you can find reference articles that I have used in various stages</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker?view=sql-server-ver15&amp;pivots=cs1-bash">https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker?view=sql-server-ver15&amp;pivots=cs1-bash</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver15">https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver15</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><a href="https://github.com/mkleehammer/pyodbc/wiki/Connecting-to-SQL-Server-from-Linux">https://github.com/mkleehammer/pyodbc/wiki/Connecting-to-SQL-Server-from-Linux</a></p>
<!-- /wp:paragraph -->