# Database API #

Internally, Kamailio uses a reduced set of SQL operations to access the records on the storage system. This allowed to write DB driver modules 
for non-SQL storage systems, such as **db_text** -- a tiny DB engine using text files.

Therefore, the interface provides data types and functions that are independent
of underlying DB storage. A DB driver module has to implement the functions specified
by the interface and provide a function named **db_bind_api(...)**
to link to the interface functions.

Starting with version 3.0.0, Kamailio has two variants of database APIs, stored
as internal libraries, named **srdb1** and **srdb2**. Most of Kamailio modules are using
**srdb1**, thus for this document the focus will be on how to use this option.

The DB1 interface is implemented in the **lib/srdb1** directory. To
use it, one has to include the file [**lib/srdb1/db.h**](https://github.com/kamailio/kamailio/blob/5.3/src/lib/srdb1/db.h).

# DB1 API Structure #

It is the structure that gets filled when binding to a DB driver module. It links to
the interface functions implemented in the module.

## Definition ##

    ...
    typedef struct db_func {
        unsigned int cap;   /* Capability vector of the database transport */
        db_use_table_fuse_table; /* Specify table name */
        db_init_finit;      /* Initialize database connection */
        db_close_f   close; /* Close database connection */
        db_query_f   query; /* query a table */
        db_fetch_result_f fetch_result;   /* fetch result */
        db_raw_query_fraw_query; /* Raw query - SQL */
        db_free_result_f  free_result;/* Free a query result */
        db_insert_f  insert;/* Insert into table */
        db_delete_f  delete;/* Delete from table */ 
        db_update_f  update;/* Update table */
        db_replace_f replace;   /* Replace row in a table */
        db_last_inserted_id_f  last_inserted_id;  /* Retrieve the last inserted ID
                                  in a table */
        db_insert_update_f insert_update; /* Insert into table, update on duplicate key */ 
        db_insert_delayed_f insert_delayed;  /* Insert delayed into table */
        db_affected_rows_f affected_rows; /* Numer of affected rows for last query */
        } db_func_t;
    ...

The attribute **cap** is a bitmask of implemented
functions, making easy to detect the capabilities of the DB driver module. A module
using the DB API should check at startup that the DB driver configured to be used
has the required capabilities. For example, **msilo**
module need **select**, **delete** and **insert** capabilities. The flags for capabilities
are enumerated in the next figure (located in **lib/srdb1/db_cap.h**).

    ...
    typedef enum db_cap {
        DB_CAP_QUERY =1 &lt;&lt; 0,  /* driver can perform queries                            */
        DB_CAP_RAW_QUERY = 1 &lt;&lt; 1,  /* driver can perform raw queries                   */
        DB_CAP_INSERT =1 &lt;&lt; 2,  /* driver can insert data                               */
        DB_CAP_DELETE =1 &lt;&lt; 3,  /* driver can delete data                               */
        DB_CAP_UPDATE =1 &lt;&lt; 4,  /* driver can update data                               */
        DB_CAP_REPLACE =   1 &lt;&lt; 5,  /* driver can replace (also known as INSERT OR UPDATE) data  */
        DB_CAP_FETCH   =   1 &lt;&lt; 6,  /* driver supports fetch result queries             */
        DB_CAP_LAST_INSERTED_ID = 1 &lt;&lt; 7,  /* driver can return the ID of the last insert operation   */
        DB_CAP_INSERT_UPDATE = 1 &lt;&lt; 8,  /* driver can insert data into database, update on duplicate  */
        DB_CAP_INSERT_DELAYED = 1 &lt;&lt; 9, /* driver can do insert delayed                 */
        DB_CAP_AFFECTED_ROWS = 1 &lt;&lt; 10  /* driver can return number of rows affected by last query */
    } db_cap_t;
    ...

    #DB1 API Functions#


# Function init(...) #

Parse the DB URL and open a new connection to database.

    ...
    typedef db1_con_t* (*db_init_f) (const str* _url);
    ...

Parameters:

* **_url** - database URL. Its format depends on DB driver. For an SQL server like MySQL has to be:
**mysql://username:pa  word@server:port/database**.
For **db_text** has to be: **text:///path/to/db/directory**.

The function returns pointer to db1_con_t* representing the connection structure or NULL
in case of error.


# Function close(...) #

The function closes previously open connection and frees all previously allocated
memory. The function db_close must be the very last function called.

## Function type ##

    ...
    typedef void (*db_close_f) (db1_con_t* _h);
    ...

Parameters:

* **_h** - db1_con_t structure representing the database connection.

The function returns nothing.

# Function use_table(...) #

Specify table name that will be used for subsequent operations (insert, delete, update, query).

## Function type ##

    ...
    typedef int (*db_use_table_f)(db1_con_t* _h, const str * _t);
    ...

Parameters:

* **_h** - database connection handle.
* **_t** - table name.

The function 0 if everything is OK, otherwise returns value negative value.

## Function query(...) ##
 
Query table for specified rows. This function implements the SELECT SQL directive.

## Function type ##

    ...
    typedef int (*db_query_f) (const db1_con_t* _h, const db_key_t* _k, const db_op_t* _op,
                                const db_val_t* _v, const db_key_t* _c, const int _n, const int _nc,
                                const db_key_t _o, db1_res_t** _r);
    ...

Parameters:

* **_h** - database connection handle.
* **_k** - array of column names that will be compared and their values must match.
* **_op** - array of operators to be used with key-value pairs.
* **_v** - array of values, columns specified in _k parameter must match these values.
* **_c** - array of column names that you are interested in.
* **_n** - number of key-value pairs to match in _k and _v parameters.
* **_nc** -  number of columns in _c parameter.
* **_o** -  order by statement for query.
* **_r** -  addre   of variable where pointer to the result will be stored.

The function 0 if everything is OK, otherwise returns value negative value.

**Note:**

If _k and _v parameters are NULL and _n is zero, you will get the whole table.
If _c is NULL and _nc is zero, you will get all table columns in the result.
Parameter _r will point to a dynamically allocated structure, it is necce  ary to 
call db_free_result function once you are finished with the result.
If _op is 0, equal (=) will be used for all key-value pairs comparisons.
Strings in the result are not duplicated, they will be discarded if you call. Make 
a copy of db_free_result if you need to keep it after db_free_result.
You must call db_free_result before you can call db_query again!

# Function fetch_result(...) #

Fetch a number of rows from a result.

## Function type ##
    ...
    typedef int (*db_fetch_result_f) (const db1_con_t* _h, db1_res_t** _r, const int _n);
    ...

Parameters:

* **_h** - database connection handle.
* **_r** - structure for the result.
* **_n** - the number of rows that should be fetched.
* The function 0 if everything is OK, otherwise returns value negative value.

# Function raw_query(...) #

This function can be used to do database specific queries. Please use this function 
only if needed, as this creates portability i  ues for the different databases. 
Also keep in mind that you need to escape all external data sources that you use.
You could use the escape_common and unescape_common functions in the core for this task.

## Function type ##

    ...
    typedef int (*db_raw_query_f) (const db1_con_t* _h, const str* _s, db1_res_t** _r);
    ...

Parameters:

* **_h** - database connection handle.
* **_s** -  the SQL query.
* **_r** - structure for the result.

The function 0 if everything is OK, otherwise returns value negative value.

# Function free_result(...) #

Free a result allocated by db_query.
## Function type ##

    ...
    typedef int (*db_free_result_f) (db1_con_t* _h, db1_res_t* _r);
    ...

Parameters:

* **_h** - database connection handle.
* **_r** -  pointer to db1_res_t structure to destroy.
* The function 0 if everything is OK, otherwise returns value negative value.

# Function insert(...) #

 Insert a row into the specified table.

## Function type ##

    ...
    typedef int (*db_insert_f) (const db1_con_t* _h, const db_key_t* _k,
                                const db_val_t* _v, const int _n);
    ...

Parameters:
			
* **_h** - database connection handle.
* **_k** -   array of keys (column names).
* **_v** -   array of values for keys specified in _k parameter.
* **_n** -  number of keys-value pairs int _k and _v parameters.

The function 0 if everything is OK, otherwise returns value negative value.

# Function delete(...) #

Delete a row from the specified table.

## Function type ##
    ...
    typedef int (*db_delete_f) (const db1_con_t* _h, const db_key_t* _k, const db_op_t* _o,
                                const db_val_t* _v, const int _n);
    ...

Parameters:

* **_h** - database connection handle.
* **_k** -  array of keys (column names) that will be matched.
* **_o** -   array of operators to be used with key-value pairs.
* **_v** -array of values that the row must match to be deleted.
* **_n** -  number of keys-value pairs int _k and _v parameters.

The function 0 if everything is OK, otherwise returns value negative value.

# Function update(...) #

Update some rows in the specified table.

## Function type ##
    ...
    typedef int (*db_update_f) (const db1_con_t* _h, const db_key_t* _k, const db_op_t* _o,
                                const db_val_t* _v, const db_key_t* _uk, const db_val_t* _uv,
                                const int _n, const int _un);
    ...
Parameters:

* **_h** - database connection handle.
* **_k** -   array of keys (column names) that will be matched.
* **_o** -   array of operators to be used with key-value pairs.
* **_v** -   array of values that the row must match to be modified.
* **_uk** -   array of keys (column names) that will be modified.
* **_uv** -   new values for keys specified in _k parameter.
* **_n** -  number of key-value pairs in _v parameters.
* **_un** -  number of key-value pairs in _uv parameters.

The function 0 if everything is OK, otherwise returns value negative value.

# Function replace(...) #

Insert a row and replace if one already exists.

## Function type ##
    ...
    typedef int (*db_replace_f) (const db1_con_t* handle, const db_key_t* keys,
                                 const db_val_t* vals, const int n);
    ...

Parameters:

* **_h** - database connection handle.
* **_k** -   array of keys (column names).
* **_v** -   array of values for keys specified in _k parameter.
* **_n** -  number of keys-value pairs int _k and _v parameters.
* The function 0 if everything is OK, otherwise returns value negative value.

# Function last_inserted_id(...) #

 Retrieve the last inserted ID in a table.

## Function type ##
    ...
    typedef int (*db_last_inserted_id_f) (const db1_con_t* _h);
    ...

Parameters:

**_h** -  structure representing database connection

The function returns the ID as integer or returns 0 if the previous statement
does not use an AUTO_INCREMENT value.

# Function insert_update(...) #

Insert a row into specified table, update on duplicate key.

# Function type #

    ...
    typedef int (*db_insert_update_f) (const db1_con_t* _h, const db_key_t* _k,
                                       const db_val_t* _v, const int _n);
    ...

Parameters:

* **_h** - database connection handle.
* **_k** -   array of keys (column names).
* **_v** -   array of values for keys specified in _k parameter.
* **_n** -  number of keys-value pairs int _k and _v parameters.

The function 0 if everything is OK, otherwise returns value negative value.

# Function insert_delayed(...) #

Insert delayed a row into specified table - don't wait for confirmation from database server.

# Function type #

    ...
    typedef int (*db_insert_delayed_f) (const db1_con_t* _h, const db_key_t* _k,
                                     const db_val_t* _v, const int _n);
    ...

Parameters:

* **_h** - database connection handle.
* **_k** -   array of keys (column names).
* **_v** -   array of values for keys specified in _k parameter.
* **_n** -  number of keys-value pairs int _k and _v parameters.

The function 0 if everything is OK, otherwise returns value negative value.

# Function affected_rows(...) #

Retrieve the number of affected rows by last operation done to database.

## Function type ##

   ...
   typedef int (*db_affected_rows_f) (const db1_con_t* _h);
   ...

Parameters:

* **_h** -  structure representing database connection

The function returns the number of affected rows by previous DB operation.

## DB API Data Types ##
# Type db_key_t #

This type represents a database key (column). Every time you need to specify
key value, this type should be used.

# Definition #

    ...
    typedef str* db_key_t;
    ...

## Type db_op_t ##

This type represents an expre  ion operator uses for SQL queries.

## Definition ##

    ...
    typedef const char* db_op_t;
    ...

Predefined operators are:

# DB Expression Operators #

    ...
    typedef const char* db_op_t;
    /** operator le   than */
    #define OP_LT  "&lt;"
    /** operator greater than */
    #define OP_GT  "&gt;"
    /** operator equal */
    #define OP_EQ  "="
    /** operator le   than equal */
    #define OP_LEQ "&lt;="
    /** operator greater than equal */
    #define OP_GEQ "&gt;="
    /** operator negation */
    #define OP_NEQ "!="
    ...

## Type db_type_t ##

Each cell in a database table can be of a different type. To distinguish 
among these types, the db_type_t enumeration is used. Every value of the
enumeration represents one datatype that is recognized by the database API. 

# Definition #
    ...
    typedef enum {
        DB1_INT,   /* represents an 32 bit integer number */
        DB1_BIGINT,/* represents an 64 bit integer number */
        DB1_DOUBLE,/* represents a floating point number  */
        DB1_STRING,/* represents a zero terminated const char* */
        DB1_STR,   /* represents a string of 'str' type   */
        DB1_DATETIME,   /* represents date and time   */
        DB1_BLOB,  /* represents a large binary object*/
        DB1_BITMAP /* an one-dimensional array of 32 flags*/
    } db_type_t;
    ...

## Type db_val_t ##

This structure represents a value in the database. Several datatypes are
recognized and converted by the database API. These datatypes are 
automatically recognized, converted from internal database representation 
and stored in the variable of corresponding type.
Modules that want to use this values needs to copy them to another memory
location, because after the call to free_result there are not more available.
If the structure holds a pointer to a string value that needs to be freed 
because the module allocated new memory for it then the free flag must be
set to a non-zero value. A free flag of zero means that the string data must
be freed internally by the database driver.

# Definition #

    ...
    typedef struct {
        db_type_t type; /* Type of the value                */
        int nul;		/* Means that the column in database has no value */
        int free;		/* Means that the value should be freed */
        /** Column value structure that holds the actual data in a union.  */
        union {
            int int_val;/* integer value              */
            long longll_val;/* long long value        */
            double   double_val; /* double value      */
            time_t   time_val;   /* unix time_t value */
            const char*   string_val; /* zero terminated string*/
            str str_val;/* str type string value      */
            str blob_val;   /* binary object data     */
            unsigned int  bitmap_val; /* Bitmap data type */
        } val;
    } db_val_t;
    ...

# Type db_con_t #

This structure represents a database connection, pointer to this structure
are used as a connection handle from modules uses the db API.

## Definition ##

    ...
    typedef struct {
        const str* table; /* Default table that should be used     */
        unsigned long tail;/* Variable length tail, database module specific */
    } db1_con_t;
    ...

Structure holding the result of a query table function. It represents one
row in a database table. In other words, the row is an array of db_val_t
variables, where each db_val_t variable represents exactly one cell in the
table.

    ...
    typedef struct db_row {
        db_val_t* values;  /* Columns in the row */
        int n;   /* Number of columns in the row */
    } db_row_t;
    ...

## Type db1_res_t ##
This type represents a result returned by db_query function (see below).
The result can consist of zero or more rows (see db_row_t description).

**Note:** A variable of type db_res_t
returned by db_query function uses dynamically allocated memory, don't forget
to call db_free_result if you don't need the variable anymore. You will
encounter memory leaks if you fail to do this! In addition to zero or more
rows, each db_res_t object contains also an array of db_key_t objects. The
objects represent keys (names of columns). 

## Definition ##
    ...
    typedef struct db1_res {
        struct {
            db_key_t* names;   /* Column names      */
            db_type_t* types;  /* Column types      */
            int n;   /* Number of columns           */
        } col;
        struct db_row* rows;   /* Rows              */
        int n;   /* Number of rows in current fetch */
        int res_rows;/* Number of total rows in query   */
        int last_row;/* Last row                    */
        } db1_res_t;
    ...

# Macros #

The DB API offers a set of macros to make easier to acce   the attributes in various data
structures.

Macros for **db_res_t**:

* RES_NAMES(res) - get the pointer to column names
* RES_COL_N(res) - get the number of columns
* RES_ROWS(res) - get the pointer to rows
* RES_ROW_N(res) - get the number of rows

Macros for **db_val_t**:

* ROW_VALUES(row) - get the pointer to values in the row
* ROW_N(row) - get the number of values in the row

Macros for **db_val_t**:

* VAL_TYPE(val) - get/set the type of a value
* VAL_NULL(val) - get/set the NULL flag for a value
* VAL_INT(val) - get/set the integer value
* VAL_BIGINT(val) - get/set the big integer value
* VAL_STRING(val) - get/set the null-terminated string value
* VAL_STR(val) - get/set the str value
* VAL_DOUBLE(val) - get/set the double value
* VAL_TIME(val) - get/set the time value
* VAL_BLOB(val) - get/set the blob value

## Example of usage ##

A simple example of doing a select. The table is named test and has two columns.

    ...
    create table test (
        a int,
        b varchar(64)
    );
    ...

The C code:
    ...
    <![CDATA[
    #include "../../dprint.h"
    #include "../../lib/srdb1/db.h"
    
    db_func_t db_funcs; /* Database API functions */
    db1_con_t* db_handle=0;   /* Database connection handle */

    int db_example(char *db_url)
    {
        str table;
        str col_a;
        str col_b;
        int nr_keys=0;
        db_key_t db_keys[1];
        db_val_t db_vals[1];
        db_key_t db_cols[1];
        db1_res_t* db_res = NULL;
        
        /* Bind the database module */
        if (db_bind_mod(&db_url, &db_funcs))
        {
            LM_ERR("failed to bind database module\n");
            return -1;
        }
        /* Check for SELECT capability */
        if (!DB_CAPABILITY(db_funcs, DB_CAP_QUERY))
        {
            LM_ERR("Database modules does not "
            "provide all functions needed here\n");
            return -1;
        }
        /* Connect to DB */
        db_handle = db_funcs.init(&db_url);
        if (!db_handle)
        {
            LM_ERR("failed to connect database\n");
            return -1;
        }
        
        /* Prepare the data for the query */
        table.s = "test";
        table.len = 4;
        col_a.s = "a";
        col_a.len = 1;
        col_b.s = "b";
        col_b.len = 1;
        
        db_cols[0] = &col_b;
        db_keys[0] = &col_a;
        
        db_vals[nr_keys].type = DB_INT;
        db_vals[nr_keys].nul = 0;
        db_vals[nr_keys].val.int_val = 1;
        nr_keys++;
        
        /* execute the query */
        /* -- select b from test where a=1 */
        db_funcs.use_table(db_handle, &table);
        if(db_funcs.query(db_handle, db_keys, NULL, db_vals, db_cols,
        nr_keys /*no keys*/, 1 /*no cols*/, NULL, &db_res)!=0)
        {
            LM_ERR("failed to query database\n");
            db_funcs.close(db_handle);
            return -1;
        }
        
        if (RES_ROW_N(db_res)<=0 || RES_ROWS(db_res)[0].values[0].nul != 0)
        {
            LM_DBG("no value found\n");
            if (db_res!=NULL && db_funcs.free_result(db_handle, db_res) < 0)
                LM_DBG("failed to free the result\n");
                db_funcs.close(db_handle);
                return -1;
        }
        
        /* Print the first value */
        if(RES_ROWS(db_res)[0].values[0].type == DB_STRING)
            LM_DBG("first value found is [%s]\n",
            (char*)RES_ROWS(db_res)[0].values[0].val.string_val);
        else if(RES_ROWS(db_res)[0].values[0].type == DB_STR)
            LM_DBG("first value found is [%.*s]\n",
            RES_ROWS(db_res)[0].values[0].val.str_val.len,
            (char*)RES_ROWS(db_res)[0].values[0].val.str_val.s);
        else
            LM_DBG("first value found has an unexpected type [%d]\n",
            RES_ROWS(db_res)[0].values[0].type);
        
        /* Free the result */
        db_funcs.free_result(db_handle, db_res);
        db_funcs.close(db_handle);
        
        return 0;
        }
    ]]>
    ...
