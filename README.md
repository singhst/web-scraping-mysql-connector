### web-scraping-tv-movie

Modules to handle the data (mainly Movie related) scraped from www.reelgood.com. 

Perfrom CRUD using Python.

| Operation | Module name |
| --------- | ----------- |
| Set up db | `localConfig.env`, `setupDatabase.py`, `checkCreateTable.py` |
| Create | `insert.py` |
| Read | `readTable.py` |
| Update | `update.py` |
| Delete | `[xxx]` |


## Usage

Make sure the configurations inside `localConfig.env` are correct. 

Example:

```
[DB]
username = root
password = 00000000
host = localhost
port = 3306
```


1. Initialize the `class` in `setupDatabase.py`. 
    ```python
    db = setupDatabase(db_name=get_movies_or_tv, db_table_name=db_table_name)
    db_connection = db.getConnection()
    ```


2. Create a database/table in MySQL server.

    Inside `setupDatabase()` class, class method `checkCreateDatabase()` checks the existance of database. Create it if not exists. 

    ```python
    def checkCreateDatabase(self):
        print(f'mysql> Created database `{self.database}`... ', end='')
    
        # executing cursor with execute method and pass SQL query
        sql_query = f"CREATE DATABASE IF NOT EXISTS {self.database};"
        self.db_cursor.execute(sql_query)
    
        print(f'==> Done!')
    ```

    `checkCreateTable()` function placed inside `checkCreateTable.py`, this function checks the existance of table, and create it if not exists. 
    ```python
    def checkCreateTable(db_connection,
                         db_table_name: str) -> bool:
        print(f'mysql> Creating table `{db_table_name}` in `{db_connection.database}` database... ', end='')
    
        # creating database_cursor to perform SQL operation
        db_cursor = db_connection.cursor()
        
        # sql query
        #"rg_id": "55a2e378-dfb0-4473-b105-7478bb1dcfc1",
        sql_query = f'''
            CREATE TABLE IF NOT EXISTS {db_table_name} (
                id INT NOT NULL AUTO_INCREMENT,
                scraped_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                rg_id VARCHAR(50) NOT NULL, 
                title VARCHAR(255), 
                year INT, 
                overview VARCHAR(512),
                rating VARCHAR(10), 
                imdb_score VARCHAR(10),     
                reelgood_rating_score VARCHAR(10),
                url_offset_value INT,
                PRIMARY KEY(id, rg_id)
            );
        '''
    
        try:
            db_cursor.execute(sql_query)
            print('==> Done!')
    
        except(Exception, mysqlError) as error:
            print(f'\n\t==> Fail.')
            print(f'\t> Error = `{error}`')
    ```


3. Perform `Create`, `Read`, `Update` and `Delete`

    - `Create`, Insert rows to a table. 
        Module name:    `insert.py`

        There are three types of inserting method:
        1. Insert single row with separating var
           
            ```python
            def insertARowToDb(db_connection,
                               table_name: str = '',
                               rg_id: str = '',
                               title: str = '',
                               year: str = '',
                               overview: str = '',
                               rating: str = '',
                               imdb_score: str = '',
                               reelgood_rating_score: str = '',
                               url_offset_value: str = '-1',
                               close_connection_afterward: bool = True) -> int:
            ```
            
        2. Insert multile rows in `List[tuple]` data type

            ```python
            def insertNRowsToDb(db_connection,
                                table_name: str,
                                record: List[tuple],
                                close_connection_afterward: bool = True) -> int:
            ```

        3. Insert multile rows in `pandas.DataFrame` data type

            ```python
            def insertPandasDfToDb(db_connection,
                                   table_name: str,
                                   df: pd.DataFrame,
                                   close_connection_afterward: bool = True) -> int:
            ```

        The below is the function to directly deal with SQL query,
        
        <details>
        <summary>`_tryAddRecordToDb()`...</summary>
        
        ```python
        def _tryAddRecordToDb(db_connection,
                              table_name: str,
                              record: List[tuple],
                              close_connection_afterward: bool) -> int:
            """
            Private function. Add record(s) to MySQL database. 
        
                Args:
                    db_connection:  `(class) MySQLConnection`, Connection to a MySQL Server
                    table_name:     `str`, the table you want to insert data in
                    record:         `List[tuple]`, data to save into database
                                    e.g. `[(data1, data2, ...), (...), ...]`
                    close_connection_afterward:     `bool`, default `True`. Choose to close `cursor` and `mysql connection` after operation.
            
                Returns:
                    int: no. of rows added to database
            """
            db_cursor = db_connection.cursor()
            
            sql_query = f'''
                INSERT INTO {table_name} (
                    rg_id,
                    scraped_timestamp,
                    title, 
                    year, 
                    overview,
                    rating, 
                    imdb_score, 
                    reelgood_rating_score,
                    url_offset_value
                    ) 
                VALUES''' + '(%s, NOW(), %s, %s, %s, %s, %s, %s, %s);'
            
            added_row_count = 0
            
            old_row_count = getRecordsCount(db_cursor, table_name)
            
            db_cursor.executemany(sql_query, record)
            db_connection.commit()
            
            curr_row_count = getRecordsCount(db_cursor, table_name)
            
            added_row_count = curr_row_count - old_row_count
            
            print(f'==> Done!')
            print(f'mysql> {added_row_count} Record inserted successfully into `{table_name}` table, {old_row_count}th-row to {curr_row_count}th-row')
            
            if close_connection_afterward:
                db_cursor.close()
                if db_connection.is_connected():
                    db_connection.close()
                    print('mysql>>> MySQL connection is closed\n')
            
            return added_row_count
        ```
        </details>


    - `Read`, Read all rows from a table.
        Module name:    `readTable.py`
        
        <details>
        <summary>`readTableAll()`...</summary>
        
        ```python
        def readTableAll(db_connection,
                         table_name: str,
                         close_connection_afterward: bool = True) -> List[tuple]:
            '''
            Public function. Query full table data. 
                Args
                ---
                    db_connection:  `(class) MySQLConnection`, Connection to a MySQL Server
                    table_name:     `str`, the table you want to insert data in
                    close_connection_afterward:     `bool`, default `True`. Choose to close `db_cursor` and `mysql connection` after operation.
        
                Queried result from MySQL
                ---
                    `List[tuple]`:  e.g. `[(id, column1, column2, ...), (...), ...]`
        
                Return
                ---
                    `Iterable[dict]`, a `list of dict` in json format.
                    e.g.
                    [
                        {'Title': 'Breaking Bad', 'Year': 2008},
                        {'Title': 'Game of Thrones', 'Year': 2011},
                        ...
                    ]
        
                Remark
                ------
                    `json.dumps(the_return_dict_list)` make return dict become JSON string.
            '''
        
            print(f'mysql> Reading records from `{table_name}` table in `{db_connection.database}` database... ', end='')
        
            # creating a db_cursor to perform a sql operation
            # returns dict list if `dictionary is True` ==> https://dev.mysql.com/doc/connector-python/en/connector-python-api-mysqlconnection-cursor.html
            db_cursor = db_connection.cursor(dictionary=True)
        
            # sql query
            query = f'''SELECT * FROM {table_name};'''
        
            record = None
        
            try:
                count = getRecordsCount(cursor=db_cursor, table_name=table_name)
                if count == 0:
                    print(f'\n\t==> Fail.')
                    print(f'\tmysql> No data present in `{table_name}` table in `{db_connection.database}` database.')
                else:
                    # execute the command
                    db_cursor.execute(query)
                    db_cursor
                    record = db_cursor.fetchall()
                    print(f'==> Done!')
                    
            except(Exception, mysqlError) as error:
                print(f'\n\t==> Fail.')
                print(f'\t> Error = `{error}`')

            if close_connection_afterward:
                if db_connection.is_connected():
                    db_cursor.close()
                    db_connection.close()
                    print('mysql>>> MySQL connection is closed\n')

            return record #json.dumps(record, indent=4, sort_keys=True, default=str)
        ```
        </details>


    - `Update`, Update a row (columns: `rg_id`, `overview` etc.) by ID from a table.
        Module name:    `update.py`

        <details>
        <summary>`updateRowById()`...</summary>
        
        ```python
        def updateRowById(db_connection,
                        table_name: str,
                        eid: str,
                        title: str,
                        rg_id: str,
                        overview: str,
                        close_connection_afterward: bool = True) -> List[tuple]:
            """
            Public function. Query full table data. 

                Args:
                    db_connection:  `(class) MySQLConnection`, Connection to a MySQL Server
                    table_name:     `str`, the database table you want to insert data to 
                    title:          `str`, 
                    rg_id:          `str`, ID used by Reelgoog
                    overview:       `str`, description of the movie/TV show
                    close_connection_afterward:     `bool`, default `True`. Choose to close `cursor` and `mysql connection` after operation.

                Returns:
                    `List[tuple]`:  Data queried from database.
                                    e.g. `[(id, column1, column2, ...), (...), ...]`
            """
            print(f'mysql> Updating id = `{eid}`, title = `{title}` in  `{table_name}` table in `{db_connection.database}` database... ', end='')
            
            # creating a cursor to perform a sql operation
            db_cursor = db_connection.cursor()

            # sql query
            query = f'''UPDATE {table_name} SET rg_id = %s, overview = %s WHERE id = %s AND title = %s;'''

            try:
                record = get_by_id(cursor=db_cursor, table_name=table_name, eid=eid)
                if record is None:
                    print(f'\n\t==> Fail.')
                    print(f'\t> Movie id = `{eid}`, title = `{title}` not found')
                else:
                    # execute the command
                    db_cursor.execute(query, [rg_id, overview, eid, title])
                    # commit the changes
                    db_connection.commit()

                    print('==> Done!')
            except(Exception, mysqlError) as error:
                print(f'\n\t==> Fail.')
                print(f'\t> Error = `{error}`')

                # if scraped string in `overview` column is too long, increase its Column Size.
                if error.errno == errorcode.ER_DATA_TOO_LONG:
                    start_index = error.msg.find('column \'') + len('column \'')
                    end_index = error.msg.find('\' at')
                    column_name = error.msg[start_index:end_index]
                    print(f'\t> column_name = error.msg[start_index:end_index] = `{column_name}`.')
                    updateColumnSize(db_connection=db_connection, table_name=table_name, column_name=column_name, size=len(overview))
                    #recursion, try update the row again
                    updateRowById(db_connection,
                                table_name,
                                eid,
                                title,
                                rg_id,
                                overview,
                                close_connection_afterward)
            finally:
                if close_connection_afterward:
                    if db_connection is not None:
                        db_cursor.close()
                        db_connection.close()
                        print('mysql>>> MySQL connection is closed\n')
        ```
        </details>

    - `Delete`