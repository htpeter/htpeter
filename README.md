### Hi there, I am Peter aka @htpeter 👋

I not have have anything to share currently. Please feel free to read some of my projects although many are for my own use and do not have the necessary public READMEs. 

If you use postgres for data science, I would love to share with you my below Postgres class that manages a lot of the messiness of managing connections but lets you perform the necessary functions. 
```{python}
"""
A high level database object that manages connection details.

Example:

credentials = {
  "host" : "a.b.com",
  "port" : 5432,
  "user" : "htpeter",
  "dbname" : "prod"
}

# connect to database 
a_database = Postgres(crdentials = credentials, connect = True)

# list of dictionaries where each item is a row with your query result
a_database.get_query_results(sql: str)

# a generator of a list of dictionaries where each item is a row with your query result
a_database.get_query_iterator(sql: str)

# returns a dataframe of your query response
a_database.get_dataframe_from_query(sql: str)

# commits a query (useful for inserts, updates, deletes, etc.)
a_database.commit_query(sql: str)

# writes a dataframe to a table in the database
a_database.write_dataframe_to_table(dataframe: pd.DataFrame, table: str, columns: list/tuple)
"""
from io import StringIO

import pandas as pd
import psycopg2 as pg
from psycopg2.extras import RealDictCursor

class Postgres:
    def __init__(self, credentials, connect=False):
        self.creds = credentials
        self.host = self.creds["host"]
        self.port = self.creds["port"]
        self.user = self.creds["user"]
        self.dbname = self.creds["dbname"]

        if connect == True:
            self._connect()

    def _connect(self):
        self.conn = pg.connect(
            host=self.host,
            port=self.port,
            dbname=self.dbname,
            user=self.user,
            password=self.password,
            cursor_factory=RealDictCursor,
        )

    def _gen_cursor(self):
        try:
            self.cur = self.conn.cursor()
        except:
            try:
                self.conn.rollback()
                self.cur = self.conn.cursor()
            except:
                self._connect()

    def _connection_health_check(self):
        if not hasattr(self, "conn"):
            self._connect()
        # if conn.closed == 0, its still open
        if self.conn.closed == 0:
            pass
        else:
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
        try:
            (conn_status, conn_details) = self._connection_health_check()
        except:
            self._connect()
            (conn_status, conn_details) = self._connection_health_check()

        self.conn.rollback()
        try:
            self.cur.execute(sql)
        except:
            self._connect()
            self.conn.rollback()
            self.cur.execute(sql)
        if commit == True:
            self.cur.commit()

        if fetch_type == "all":
            return self.cur.fetchall()
        elif fetch_type == "iterator":
            return self.cur

    def get_query_results(self, sql):
        result_object = self._execute(sql, commit=False, fetch_type="all")
        return result_object

    def get_query_iterator(self, sql):
        cursor = self._execute(sql, commit=False, fetch_type="iterator")
        return cursor

    def get_dataframe_from_query(self, sql):
        return pd.DataFrame(self.get_query_results(sql))

    def commit_query(self, sql):
        result_object = self._execute(sql, commit=True, fetch_type="all")
        return result_object

    def close(self):
        self.conn.close()

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
            try:
                (conn_status, conn_details) = self._connection_health_check()
            except:
                self._connect()
                (conn_status, conn_details) = self._connection_health_check()
            self.conn.rollback()
            _efficent_psycopg2_write()
        except:
            _efficent_psycopg2_write()

    def __repr__(self):
        return f"""
        host : {self.host}, user : {self.user}
        """
```
