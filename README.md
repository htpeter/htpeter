## A Bit About `htpeter`

Hello, my name is Peter and I am a Data Scientist who currently lives and works in New York City. I enjoy working with teams and trying to implement USEFUL machine learning products. Nowadays we want to infuse ML into many things, but this is not the best modus operandi. We should start all ML projects with "What economic benefit do we get by improving our ability to predict here?". From this question, we can begin the process of exploring methods that will help improve a given system. This works whether that "system" is a revenue generating business or an internal engineering system.


#### What I've Been Up To Recently
I've recently been working on using NLP to solve business problems (thanks to `explosionai.spacy` and `huggingface.transformers`). I also have become interested in how less popular methods like Multi-Armed bandits can be used to build self-improving systems and how Monte Carlo simulation can be used to breakdown and analyze complex systems. 


#### My Coding & Code Examples

I mainly code in Python and SQL. I'm also a huge Unix fan (the kind who runs `i3`, `tmux` and knows their way around `vim`). I've also had a fair bit of experience writing `go`. In my freetime, I will occasionally putz around with functional languages although I have yet to enter my post-modern functional only decade where I really cut my teeth. I imagine the bug will infect me sooner rather than later.

**Here are some code examples that will give you a taste as to how I code**

###### A Python Interface for Postgres 

The below code examples and underlying python Class is how I've been connecting to Postgres over the last few years. It lets me easily return `pandas.DataFrame` objects from queries and write dataframes to tables when needed. It kicks authentication down to `.pgpass`. It has helped my fellow team members simplify their code by never having to hard code connection details.

Writing dataframes uses psycopg2's most efficent implementation which is >10X faster than iterating over each row and writing inserts. If your team is doing this, try implementing the below.

**How it is used**

In this example, we will connect to a database and calculate an aggregated view, writing the results to a table. 
```python
my_database  = Postgres({
    "host" : "127.0.0.1",
    "dbname" : "an_example_database",
    "user" : "peter",
    "port" : 5432
})

aggregated_revenue_data = my_database.get_dataframe_from_query("""
  SELECT 
    TO_CHAR(ct.transaction_date, 'YYYY-MM') as month,
    c.customer_name,
    sum(ct.value) as monthly_revenue
  FROM customer_transactions ct 
  LEFT JOIN customers c ON c.id = ct.customer_id
  GROUP BY 1, 2
""")

# CREATE table unless it already exists
my_database.commit_query("""
  CREATE TABLE IF NOT EXISTS etl.monthly_customer_revenue (
    month text,
    customer_name text,
    monthly_revenue float
  )
""")

# TRUNCATE the table if it already existed
my_database.commit_query("""
  TRUNCATE etl.monthly_customer_revenue
""")

my_database.write_dataframe_to_table(
  dataframe = aggregated_revenue_data,
  table = "etl.monthly_customer_revenue",
  columns = ("month", "customer_name", "monthly_revenue")
)
```

**How it is built**

The goal of this class is to create a simple plug and play interface to a Postgres database that offers the user all the functions a data scientist would want. Namely (1) get some data (2) update some data and (3) move some data. I aim to make these 3 things very easy while abstracting all of the `psycopg2` connection nuances like handling rollbacks after failed commits, returning the SQL level errors instead of python errors, and reconnecting from a slew of different states.

**The connection check and reconnect methods could certaintly use an update! It is important to know which part of your code sub-optimal. If I were to score the below, its biggest risk is that I don't understand psycopg2's internals well enough and therefore my connection methodology is likely not how the author of psycopg2 would do it. I hope the convenience of my abstraction makes up for it!**

```python
from io import StringIO

import psycopg2 as pg
from psycopg2.extras import RealDictCursor
import pandas as pd


class Postgres:
    """
    Some fork of Postgres expected (Redshft, Postgres)
    """

    def __init__(self, credentials, auto_connect=False):
        self.dbtype = "postgres"
        self.creds = credentials

        self.host = self.creds["host"]
        self.port = self.creds["port"]
        self.user = self.creds["user"]
        self.dbname = self.creds["dbname"]

        if auto_connect == True:
            self._connect()
        else:
            pass

    def _connect(self):
        self.conn = pg.connect(
            host=self.host,
            port=self.port,
            dbname=self.dbname,
            user=self.user,
            cursor_factory=RealDictCursor,
        )

    def _gen_cursor(self):
        try:
            self.cur = self.conn.cursor()
        except:
            self.conn.rollback()
            self.cur = self.conn.cursor()

    def _connection_health_check(self):
        # if conn.closed == 0, its still open
        try:
            if self.conn.closed == 0:
                pass
            else:
                self._connect()
        except:
            self._connect()
        # cursor exists
        if hasattr(self, "cur"):
            return (True, {"connection": "live", "cursor": "exists"})
        # create cursor if it doesn't exist but our connection is live
        else:
            self._gen_cursor()
            return (True, {"connection": "live", "cursor": "refreshed"})

    def _execute(self, sql, commit=False, fetch_type="all"):
        """
        Run health check and then executes a query.

        Arguments:
            sql <str>
                : Your SQL code to execute.
            commit <bool>
                : Determines if your SQL requires a COMMIT to be executed afterwards.
            fetch_type <str> in ["all", "interator"]
                :
        Returns
            if commit = False
                if fetch_type = "all"      : returns list of dictionaries for each row in your query result.
                if fetch_type = "iterator" : returns psycopg2 cursor which can be treated as an iterator of list of dicts.
            if commit = True
                returns boolean for success, row amount if write
        """
        (conn_status, conn_details) = self._connection_health_check()
        try:
            self.conn.rollback()
        except pg.OperationalError as e:
            # operational errors generally occur with stale connection
            self._connect()
        try:
            self.cur.execute(sql)
        except:
            # cursors can also go stale and rollback may not refresh cursor,
            # in this case generate a new cursor
            self.conn.rollback()
            self._gen_cursor()
            self.cur.execute(sql)
        if commit == True:
            self.conn.commit()

        # based on what should be returned to the user
        if commit == True:
            return None
        elif fetch_type == "all":
            return self.cur.fetchall()
        elif fetch_type == "iterator":
            return self.cur

    def get_query_results(self, sql):
        result_object = self._execute(sql, commit=False, fetch_type="all")
        return result_object

    def get_query_iterator(self, sql):
        cursor = self._execute(sql, commit=False, fetch_type="iterator")
        return cursor

    def get_dataframe_from_query(self, sql, iterative_cursor=False):
        if iterative_cursor == True:
            return pd.DataFrame(self.get_query_iterator(sql))
        else:
            return pd.DataFrame(self.get_query_results(sql))

    def commit_query(self, sql):
        result_object = self._execute(sql, commit=True, fetch_type="all")
        return result_object

    def write_dataframe_to_table(self, dataframe, table, columns):
        """
        Writes dataframe to Postgres using efficent psycopg2 stream + cursor.copy_from
        syntax.
        """
        conn_status, _ = self._connection_health_check()
        if conn_status != True:
            self._connect()

        def _efficent_psycopg2_write():
            stream = StringIO()
            dataframe.to_csv(
                stream, sep="\t", header=False, index=False, float_format="%.0f"
            )
            stream.seek(0)
            contents = stream.getvalue()
            self.cur.copy_from(stream, table, null="", columns=columns)
            self.conn.commit()

        try:
            _efficent_psycopg2_write()
        except:
            self._connection_health_check()
            _efficent_psycopg2_write()

    def close(self):
        self.conn.close()
```
