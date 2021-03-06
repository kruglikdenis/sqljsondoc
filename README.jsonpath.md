

<!--stackedit_data:&#10;eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19&#10;-->

## Introduction to SQL/JSON

SQL-2016 standard doesn't describes the JSON data type, but instead it  introduced SQL/JSON data model (not  JSON data type like XML )  with string storage and path language used by certain SQL/JSON functions to query JSON.   SQL/JSON data model is a sequences of items, each of which is consists of SQL scalar values with an additional SQL/JSON null value,  and composite data structures using JSON arrays and objects.

PostgreSQL has two JSON  data types - the textual json data type to store an exact copy of the input text and the `jsonb` data type -   the binary storage for  JSON data converted to PostgreSQL types, according  mapping in   [json primitive types and corresponding  PostgreSQL types](https://www.postgresql.org/docs/current/static/datatype-json.html).  SQL/JSON data model adds datetime type to these primitive types, but it only used for comparison operators in path expression and stored on disk as a string.  Thus, `jsonb` data is already conforms to SQL/JSON data model (ORDERED and UNIQUE KEYS), while json should be converted according the mapping.  SQL-2016 standard describes two sets of SQL/JSON functions : constructor functions and query functions.  Constructor functions use values of SQL types and produce JSON values (JSON objects or JSON arrays) represented in SQL character or binary string types. Query functions evaluate SQL/JSON path language expressions against JSON values, producing values of SQL/JSON types, which are converted to SQL types.

## SQL/JSON Path language 

The main task of the path language is to specify  the parts (the projection)  of JSON data to be retrieved by path engine for the  SQL/JSON query functions.  The language is designed to be flexible enough to meet the current needs and to be adaptable to the future use cases. Also, it is integratable into SQL engine, i.e., the semantics of predicates and operators generally follow SQL.  To be friendly to JSON users, the language resembles  JavaScript - dot(`.`)  used for member access and `[]` for array access, arrays starts from zero (SQL arrays starts from 1).

Example of two-floors house:
```sql
CREATE TABLE house(js) AS SELECT jsonb '
{
  "info": {
    "contacts": "Postgres Professional\n+7 (495) 150-06-91\ninfo@postgrespro.ru",
    "dates": ["01-02-2015", "04-10-1957 19:28:34 +00", "12-04-1961 09:07:00 +03"]
  },
  "address": {
    "country": "Russia",
    "city": "Moscow",
    "street": "117036, Dmitriya Ulyanova, 7A"
  },
  "lift": false,
  "floor": [
    {
      "level": 1,
      "apt": [
        {"no": 1, "area": 40, "rooms": 1},
        {"no": 2, "area": 80, "rooms": 3},
        {"no": 3, "area": null, "rooms": 2}
      ]
    },
    {
      "level": 2,
      "apt": [
        {"no": 4, "area": 100, "rooms": 3},
        {"no": 5, "area": 60, "rooms": 2}
      ]
    }
  ]
}
';
```
It can be illustrated as [this picture](https://ic.pics.livejournal.com/obartunov/24248903/45289/45289_original.png).

 Consider the following path expression:
```sql
'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)'
```
The result of this path expression will be information about apartments with rooms, which area is  in specified range. Dollar sign `$`  designates a __context item__ or the whole JSON document, which describes a house with floors ( `floor[]`), apartments (`apt[]`)  with rooms and room has attribute `area`.  Expression `$.floor[*].apt[*]` in described context means  __any__ room, which filtered by a filter expression (in parentheses).  At sign `@` in filter designates the __current item__ in filter.   Path expression could have more filters (applied  left to right) and each of them may be nested. 
```sql
'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90) ? (@.rooms > 2)'
```
It's possible to use the __path variables__ in path expression, whose values are set in __`PASSING`__ clause of invoked SQL/JSON function. For example (js is a column of type JSON):
```sql
SELECT JSON_QUERY(js, '$.floor[*].apt[*] ? (@.area > $min && @.area < $max)' PASSING 40 AS min, 90 AS max WITH WRAPPER) FROM house;
                               json_query
------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
```
__`WITH WRAPPER`__ clause  is used to wrap the results into array, since  `JSON_QUERY` output should be  JSON text.  If minimal and maximal values are stored in table `area (min integer, max integer)`, then it is possible to pass them to the path expression:
```sql
SELECT JSON_QUERY(
  house.js, '$.floor[*].apt[*] ? (@.area > $min && @.area < $max)'
  PASSING area.min AS min,
          area.max AS max
  WITH WRAPPER
  EMPTY ARRAY ON EMPTY
)
FROM
  house,
  (VALUES (30, 50), (70, 120), (200, 300)) area(min, max);
                               json_query                                
-------------------------------------------------------------------------
 [{"no": 1, "area": 40, "rooms": 1}]
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 4, "area": 100, "rooms": 3}]
 []
(3 rows)

```
Example of using several filters in json path expression, which returns room number (integer) and the room should satisfies the several conditions: the one is checking floor level and another - its area.
```sql
SELECT JSON_VALUE(js, '$.floor[*] ? (@.level > 1).apt[*] ? (@.area > 40 && @.area < 90).no' RETURNING int) FROM house;
 json_value
------------
          5
(1 row)
```

Path expression may  contains several  __item methods__ (out of eight predefined functions), which applies to the result of preceding path expression. For example,  item method `.double()` converts `area` into a double number.
```sql
'$.floor[0].apt[1].area.double()'
```
More complex example with __keyvalue()__ method, which outputs an array of  room numbers. 
```sql
SELECT JSON_QUERY( js , '$.floor[*].apt[*].keyvalue() ? (@.key == "no").value' WITH WRAPPER) FROM house;
   json_query
-----------------
 [1, 2, 3, 4, 5]
(1 row)
```

## JSONPATH in PostgreSQL

In PostgreSQL the SQL/JSON path language is implemented as  **`JSONPATH`**  data type - the binary representation of parsed SQL/JSON path expression to effective query JSON data.  **Path expression** is an optional  path mode (strict | lax), followed by a  path, which is a  sequence of path elements,  started from path  variable, path literal or  expression in parentheses and zero or more operators ( json accessors) .  It  is possible to specify arithmetic or boolean  (*PostgreSQL extension*) expression on the path. *Path can be enclosed in brackets to return an array similar to WITH WRAPPER clause in SQL/JSON query functions. This is a PostgreSQL extension )*.  

Examples of vaild `jsonpath`:
```sql
'$.floor'
'($+1)'
'$+1'
-- boolean predicate in path, extension
'($.floor[*].apt[*].area > 10)'
'$.floor[*].apt[*] ? (@.area == null).no'
```
`Jsonpath` could be an expression:
```sql
'$' || '.' || 'a'
```

Syntactical errors in `jsonpath` are reported
```sql
SELECT '$a. >1'::jsonpath;
ERROR:  bad jsonpath representation at character 8
DETAIL:  syntax error, unexpected GREATER_P at or near ">"
```
An [Example](#how-path-expression-works) of how path expression works.

### `Jsonpath` operators

To accelerate JSON path queries using existing indexes for `jsonb`  (GIN index using built-in  `jsonb_ops` or `jsonb_path_ops`)  PostgreSQL extends the standard with two  boolean operators for `json[b]` and `jsonpath` data types.

* `json[b] @? jsonpath` -  exists  operator, returns bool.  Check that path expression returns non-empty SQL/JSON sequence.
* `json[b] @~ jsonpath` - match operator, returns the result of boolean predicate (*PostgreSQL extension*).

```sql
SELECT js @?  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
 ?column?
----------
 t
(1 row)

SELECT js @~  '$.floor[*].apt[*].area <  20' FROM house;
 ?column?
----------
 f
(1 row)
```
`jsonb @? jsonpath` and `jsonb @~ jsonpath` are as fast as `jsonb @> jsonb`  (for equality operation),  but  `jsonpath` supports more complex expressions, for example:
```sql
SELECT count(*) FROM house WHERE   js @~ '$.info.dates[*].datetime("dd-mm-yy")  > "1945-03-09".datetime()';
 count
-------
     1
(1 row)
```
Operators exists `@?` and match `@~`  can be speeded up by GIN index using built-in `jsonb_ops` or `jsonb_path_ops` opclasses.
```sql
SELECT COUNT(*) FROM bookmarks 
WHERE jb @? '$.tags[*] ? (@.term == "NYC")';
                                       QUERY PLAN
------------------------------------------------------------------------------------------------
 Aggregate (actual time=0.529..0.529 rows=1 loops=1)
   ->  Bitmap Heap Scan on bookmarks (actual time=0.080..0.502 rows=285 loops=1)
         Recheck Cond: (jb @? '$."tags"[*]?(@."term" == "NYC")'::jsonpath)
         Heap Blocks: exact=285
         ->  Bitmap Index Scan on bookmarks_jb_path_idx (actual time=0.045..0.045 rows=285 loops=1)
               Index Cond: (jb @? '$."tags"[*]?(@."term" == "NYC")'::jsonpath)
 Planning time: 0.053 ms
 Execution time: 0.553 ms
(8 rows)

SELECT COUNT(*) FROM bookmarks 
WHERE jb @~ '$.tags[*].term == "NYC"';
                                        QUERY PLAN                                               -----------------------------------------------------------------------------------------------
Aggregate (actual time=0.930..0.930 rows=1 loops=1)
   ->  Bitmap Heap Scan on bookmarks (actual time=0.133..0.884 rows=285 loops=1)
         Recheck Cond: (jb @~ '($."tags"[*]."term" == "NYC")'::jsonpath)
         Heap Blocks: exact=285
         ->  Bitmap Index Scan on bookmarks_jb_path_idx (actual time=0.073..0.073 rows=285 loops=1)
               Index Cond: (jb @~ '($."tags"[*]."term" == "NYC")'::jsonpath)
 Planning time: 0.135 ms
 Execution time: 0.973 ms
(8 rows)
```

Query operators:

* `json[b] @* jsonpath` - set-query operator,  returns `setof json[b]`.
* `json[b] @# jsonpath` - singleton-query operator,  returns a single `json[b]`.
The results is depends on the size of the resulted SQL/JSON sequence:
--  empty sequence - returns SQL NULL;
-- single item - returns the item;
-- more items - returns an array of items.
Notice, that such behaviour differs from `WITH CONDITIONAL WRAPPER`, since the latter wraps into array a single scalar value, but not the single object or an array.

```sql
SELECT js @*  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
             ?column?
-----------------------------------
 {"no": 2, "area": 80, "rooms": 3}
 {"no": 5, "area": 60, "rooms": 2}
(2 rows)

SELECT js @#  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
                                ?column?
------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
```
Operator `@#` can be used to index `jsonb`:
```sql
CREATE INDEX bookmarks_oper_path_idx ON bookmarks USING gin((js @# '$.tags.term') jsonb_path_ops);
EXPLAIN ( ANALYZE, COSTS OFF) SELECT COUNT(*) FROM bookmarks WHERE js @# '$.tags.term' @> '"NYC"';
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Aggregate (actual time=1.136..1.136 rows=1 loops=1)
   ->  Bitmap Heap Scan on bookmarks (actual time=0.213..1.098 rows=285 loops=1)
         Recheck Cond: ((js @# '$."tags"."term"'::jsonpath) @> '"NYC"'::jsonb)
         Heap Blocks: exact=285
         ->  Bitmap Index Scan on bookmarks_oper_path_idx (actual time=0.148..0.148 rows=285 loops=1)
               Index Cond: ((js @# '$."tags"."term"'::jsonpath) @> '"NYC"'::jsonb)
 Planning time: 0.228 ms
 Execution time: 1.309 ms
(8 rows)
```

### Path modes

The path engine has two modes, strict and lax, the latter is   default, that is,  the standard tries to facilitate matching of the  (sloppy) document structure and path expression.

In __strict__ mode any structural errors (  an attempt to access a non-existent member of an object or element of an array)  raises an error (it is up to `JSON_XXX` function to actually report it, see `ON ERROR` clause).
For example: 

```sql
SELECT JSON_VALUE(jsonb '1', 'strict $.a' ERROR ON ERROR); -- returns ERROR:  SQL/JSON member not found
```
Notice,  `JSON_VALUE` function needs `ERROR ON ERROR`  to report the error  , since default behaviour  `NULL ON ERROR` suppresses error reporting and returns `null`.

In __strict__ mode  using an array accessor on a scalar value  or  object triggers error handling.
```sql
SELECT JSON_VALUE(jsonb '1', 'strict $[0]' ERROR ON ERROR);
ERROR:  SQL/JSON array not found
SELECT JSON_VALUE(jsonb '1', 'strict $.a' ERROR ON EMPTY ERROR ON ERROR);
ERROR:  SQL/JSON member not found
```

 In __lax__ mode the path engine suppresses the structural errors and  converts them to the empty SQL/JSON sequences.  Depending on `ON EMPTY` clause  the empty SQL/JSON sequences  will be  interpreted as `null` by default ( `NULL ON EMPTY`) or raise an ERROR ( `ERROR ON EMPTY`).

```sql
SELECT JSON_VALUE(jsonb '1', 'lax $.a' ERROR ON ERROR);
 json_value
------------
 (null)
(1 row)
SELECT JSON_VALUE(jsonb '1', 'lax $.a' ERROR ON EMPTY ERROR ON ERROR);
ERROR:  no SQL/JSON item
```

Also,  in __lax__ mode arrays of size 1 is interchangeable with the singleton. 

Example of automatic array wrapping in lax mode:
```sql
SELECT JSON_VALUE(jsonb '1', 'lax $[0]' ERROR ON ERROR);
 json_value
------------
 1
(1 row)
```

### Path Elements

__path literal__
~ JSON primitive types : unicode text, numeric, true, false, null

__path variable__
~ ```$``` -- context item
~ ```$var``` -- named variable, value is set in `PASSING` clause (*may be of   datetime type*)
~ ```@``` -- value of the current item in a filter
~ ```last``` - JSON last subscript of an array

__expression in parentheses__
~  ```($a + 2)```

__Path elements__ 
 ~ __member accessor__  -- `.name` or `."$name"`.  It is used to access a member of an object by key name.  If the key name does not begin with a dollar sign and meets the JavaScript rules of an Identifier, then the member name can be written in clear text, else it can be written as a character string literal. For example: `$.color, "$."color of a pen"`.  In __strict__ mode, every SQL/JSON item in the SQL/JSON sequence must be an object with a member having the specified key name. If this condition is not met, the result is an error.  In __lax__ mode, any SQL/JSON array in the SQL/JSON sequence is unwrapped. Unwrapping only goes **one deep**; that is, if there is an array of arrays, the outermost array is unwrapped, leaving the inner arrays alone.

 
 ~ __wildcard member accessor__  --  `.*`, the values of all attributes of the current  object.  In __strict__ mode, every SQL/JSON item in the SQL/JSON sequence must be an object. If this condition is not met, the result is an error. In __lax__ mode, any SQL/JSON array in the SQL/JSON sequence is unwrapped.
 
 ~ __array element accessor__  -- `[1,5 to LAST]`, the second and the six-th to the last array elements of the array . Since jsonpath follows javascript conventions rather than SQL, `[0]` accesses is the first element in the array. `LAST` is the special variable to handle arrays of unknown length, its value is the size of the array minus 1.   In the __strict__ mode, subscripts must be singleton numeric values between 0 and last; in __lax__ mode, any subscripts that are out of bound are simply ignored. In both __strict__ and __lax__ mode, non-numeric subscripts are an error.
 
 ~ __wildcard array element accessor__ -- `[*]`, all array elements. In __strict__ mode, the operand must be an array, in __lax__ mode, if the operand is not an array, then one is provided by wrapping it in an array before unwrapping, `$[*]` is the same as `$[0 to last]`.  The latter is not valid in __strict__ mode, since `$[0 to last]` requires at least one array element and raise an error if `$` is the empty array , while `$[*]`  returns `null` in that case.
 ~ __filter expression__ --  ? (expression) , the result of filter expression may be `unknown`, `true`,  `false`. 
 ~ __item method__ -- is the function, that operate on an SQL/JSON item and return an SQL/JSON item. 
It denotes as  `.` and could be one of the 8 methods:

  ~ __type()__ - returns a character string that names the type of the SQL/JSON item ( `"null"`, `"boolean"`, `"number"`, `"string"`, `"array"`, `"object"`, `"date"`, `"time without time zone"`, `"time with time zone"`, `"timestamp without time zone"`, `"timestamp with time zone"`).
   
  ~ __size()__ -  returns the size of an SQL/JSON item, which is the number of elements in the array or 1 for SQL/JSON object or scalar in __lax__ mode  and error `ERROR:  SQL/JSON array not found` in __strict__ mode.  
```sql
SELECT JSON_VALUE('[1,2,3]', '$.size()' RETURNING int);
 json_value
------------
          3
(1 row)
```
In more complex case,  we can wrap SQL/JSON sequence into an array and apply `.size()` to the result:
```sql
 SELECT JSON_VALUE(JSON_QUERY('[1,2,3]', '$[*] ? (@ > 1)' WITH WRAPPER), '$.size()' RETURNING int);
 json_value
------------
          2
(1 row)
 ```
 Or use our automatic wrapping extension to the path expression
 ```sql
SELECT JSON_VALUE('[1,2,3]', '[$[*] ? (@ > 1)].size()' RETURNING int);
 json_value
------------
          2
(1 row)
 ```
 
   ~ __`ceiling()`__ - the same as `CEILING` in SQL
   ~ __`double()`__ - converts a string or numeric to an approximate numeric value.
   ~ __`floor()`__ - the same as `FLOOR` in SQL
   ~ __`abs()`__  - the same as `ABS` in SQL
   ~ __`datetime()`__ - converts a character string to an SQL datetime type, optionally using a conversion template ( [templates examples](https://www.postgresql.org/docs/current/static/functions-formatting.html)) . Default template is ISO - "yyyy-dd-mm".    Default timezone could be specified  as a second argument of __datetime()__ function, it applied  only if  template contains **TZH** and there is no 	timezone in input data.  That helps to keep jsonpath operators and functions to be immutable and, hence, indexable.
    
 
 PostgreSQL adds support of  conversion of UNIX epoch (double) to `timestamptz`.
```sql
SELECT JSON_VALUE('"10-03-2017"','$.datetime("dd-mm-yyyy")');
 json_value
------------
 2017-03-10
(1 row)
SELECT JSON_VALUE('"12:34:56"','$.datetime().type()');
       json_value
------------------------
 time without time zone
(1 row)

-- SQL Standard is strict about matching of data and template
-- An error occurs if some data doesn't contains timezone

SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH") ? (@ < "2000-01-01".datetime())' FROM house;
ERROR:  Invalid argument for SQL/JSON datetime function

-- Identify "problematic" dates (no TZH specified)
SELECT js @* '$.info.dates[*] ? ((@.datetime("dd-mm-yy hh24:mi:ss TZH") < "2000-01-01".datetime()) is unknown)' FROM house;
   ?column?
--------------
 "01-02-2015"
(1 row)

-- Add default timezone
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss TZH","+3") ? (@ < "2000-01-01".datetime())' FROM house;
          ?column?
-----------------------------
 "1957-10-04T19:28:34+00:00"
 "1961-04-12T09:07:00+03:00"
(2 rows)

-- datetime cannot compared to string
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss") ? (@ < "2000-01-01")' FROM house;
 ?column? 
----------
(0 rows)
-- rejected items in previous query
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss") ? ((@ < "2000-01-01") is unknown)' FROM house;
       ?column?
-----------------------
 "2015-02-01T00:00:00"
 "1957-10-04T19:28:34"
 "1961-04-12T09:07:00"
(3 rows)
-- 
SELECT js @* '$.info.dates[*].datetime("dd-mm-yy hh24:mi:ss") ? (@ > "1961-04-12".datetime())' FROM house;
       ?column?
-----------------------
 "2015-02-01T00:00:00"
 "1961-04-12T09:07:00"
(2 rows)

SELECT js @* '$.info.dates[*].datetime("dd-mm-yy") ? (@ > "1961-04-12".datetime())' FROM house;
   ?column?
--------------
 "2015-02-01"
(1 row)

-- UNIX epoch to timestamptz
SELECT jsonb '0' @* '$.datetime().type()';
          ?column?
----------------------------
 "timestamp with time zone"
(1 row)

SELECT jsonb '[0, 123.45]' @* '$.datetime()';
            ?column?            
--------------------------------
 "1970-01-01T00:00:00+00:00"
 "1970-01-01T00:02:03.45+00:00"
(2 rows)
```
   ~ __keyvalue()__ - transforms json to an SQL/JSON sequence of objects with a known schema. Example:
   ```sql
   SELECT JSON_QUERY( '{"a": 123, "b": 456, "c": 789}', '$.keyvalue()' WITH WRAPPER);
                                       ?column?
--------------------------------------------------------------------------------------
 [{"key": "a", "value": 123}, {"key": "b", "value": 456}, {"key": "c", "value": 789}]
(1 row)
```

 **PostgreSQL extensions**:
   1.  __recursive wildcard member accessor__ -- `.**`,  recursively applies wildcard member accessor `.*` to all levels of hierarchy and returns   the values of **all** attributes of the current object regardless of the level of the hierarchy.  It is possible to specify a level or range of the tree to unwrap -- `.**{2}`, `.**{2 TO LAST}`.
Examples:

```sql
-- Wildcard member accessor returns the values of all elements without looking deep.
SELECT jsonb '{"a":{"b":[1,2]}, "c":1}' @* '$.*';
   ?column?
---------------
 {"b": [1, 2]}
 1
(2 rows)

--  Recursive wildcard member accessor "unwraps"  all objects and arrays
SELECT jsonb '{"a":{"b":[1,2]}, "c":1}' @* '$.**';
           ?column?
------------------------------
 {"a": {"b": [1, 2]}, "c": 1}
 {"b": [1, 2]}
 [1, 2]
 1
 2
 1
(6 rows)

-- Specify range 
SELECT jsonb '{"a":{"b":[1,2]}, "c":1}' @* '$.**{2 to last}';
 ?column?
----------
 [1, 2]
 1
 2
(3 rows)
```
 
  2.   __automatic wrapping__  - `[path]` is equivalent to `WITH WRAPPER`  clause in `JSON_QUERY`.
   ```sql
   SELECT JSON_QUERY('[1,2,3]', '[$[*]]');
 json_query
------------
 [1, 2, 3]
(1 row)

SELECT JSON_QUERY('[1,2,3]', '$[*]' WITH WRAPPER);
 json_query
------------
 [1, 2, 3]
(1 row)
```

   
 ### Filter expression
A filter expression is similar to a `WHERE` clause in SQL, it is used to remove SQL/JSON items from an SQL/JSON sequence if they do not satisfy a predicate. The syntax uses a question mark `?` followed by a parenthesized predicate. In __lax__ mode, any SQL/JSON arrays in the operand are automatically unwrapped. The predicate is evaluated for each SQL/JSON item in the SQL/JSON sequence.  Predicate returns `Unknown` (SQL NULL) if any error occured during evaluation of its operands and execution. The result is those SQL/JSON items for which the predicate resulted in `True`, `False` and `Unknown` are rejected. 

Within a filter, the special variable `@` is used to reference the current SQL/JSON item in the SQL/JSON sequence.

The SQL/JSON path language has the following predicates:

 - `exists` predicate, to test if a path expression has a non-empty result.
- Comparison predicates `==`, `!=`, `<>`, `<`, `<=`, `>`, and `>=`.
- `like_regex` for string pattern matching.  Optional parameter`flag` can be combination of `i`, `s`, `m`, `x`, default value is `s`.
- `starts with` to test for an initial substring (prefix).
- `is unknown` to test for `Unknown` results. Its operand should be in parentheses.

JSON literals `true, false` are parsed into the SQL/JSON model as the SQL boolean values `True` and `False`. 
```sql
SELECT JSON_VALUE(jsonb 'true','$ ? (@ == true)') FROM house;
 json_value
------------
 true
(1 row)
SELECT jsonb '{"g": {"x": 2}}' @* '$.g ? (exists (@.x))';
 ?column?
----------
 {"x": 2}
(1 row)
```
JSON literal `null` is parsed into the special SQL/JSON value `null`, which differs from SQL NULL, for example, SQL JSON `null` is equal to itself, so the result of `null == null` is `True`.
```sql
SELECT JSON_QUERY('1', '$ ? (null == null)');
 json_query
------------
 1
(1 row)
SELECT JSON_QUERY('1', '$ ? (null != null)');
 json_query
------------
 (null)
(1 row)
``` 
Prefix search with `starts with` example:
```sql
SELECT js @* '$.** ? (@ starts with "11")' FROM house;
            ?column?
---------------------------------
 "117036, Dmitriya Ulyanova, 7A"
(1 row)
```
Regular expression search:
```sql
-- case insensitive
SELECT js @* '$.** ? (@ like_regex "O(w|v)" flag "i")' FROM house;
            ?column?
---------------------------------
 "Moscow"
 "117036, Dmitriya Ulyanova, 7A"
(2 rows)
-- ignore spaces in query, flag "x"
SELECT js @* '$.** ? (@ like_regex "O w|o V" flag "ix")' FROM house;
            ?column?
---------------------------------
 "Moscow"
 "117036, Dmitriya Ulyanova, 7A"
(2 rows)
-- single-line mode, flag "s".
SELECT js @* '$.** ? (@ like_regex "^info@" flag "is")' FROM house;
 ?column?
----------
(0 rows)
-- multi-line mode, flag "m"
SELECT js @* '$.** ? (@ like_regex "^info@" flag "im")' FROM house;
                             ?column?
------------------------------------------------------------------
 "Postgres Professional\n+7 (495) 150-06-91\ninfo@postgrespro.ru"
(1 row)
```


Predicate `is unknown` can be used to filter "problematic" SQL/JSON items. Notice, that filter expression automatically unwraps array `apt`, since default mode is `lax`.
```sql
SELECT js @* '$.floor.apt ? ((@.area / @.rooms > 0))' FROM house;
              ?column?
------------------------------------
 {"no": 1, "area": 40, "rooms": 1}
 {"no": 2, "area": 80, "rooms": 3}
 {"no": 4, "area": 100, "rooms": 3}
 {"no": 5, "area": 60, "rooms": 2}
(4 rows)
-- what item was rejected by a filter ?
SELECT js @* '$.floor.apt ? ((@.area / @.rooms > 0) is unknown)' FROM house;
              ?column?
-------------------------------------
 {"no": 3, "area": null, "rooms": 2}
(1 row)
```

### How path expression works

Path expression is intended to produce an SQL/JSON sequence ( an ordered list of zero or more SQL/JSON items) and completion code for  SQL/JSON query functions or operator, whose job is to process that result using the particular SQL/JSON query operator. The path engine process path expression step by step, each of which produces SQL/JSON sequence for following step. For example,  path expression  
```sql
'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)'
```
at first step produces SQL/JSON sequence of length 1, which is simply json itself - the context item, denoted as `$`.  Next step is member accessor `.floor` - the result is SQL/JSON sequence of length 1 - an array `floor`, which then unwrapped by  wildcard  array element accessor  `[*]` to SQL/JSON sequence of length 2, containing an array of two objects . Next, `.apt`  produces two arrays of objects and `[*]` extracts  SQL/JSON sequence of length 5 ( five objects - appartments), each of which filtered by a filter expression `(@.area > 40 && @.area < 90)`, so a result of the whole path expression  is a sequence of two SQL/JSON items.

```sql
SELECT JSON_QUERY(js,'$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' WITH WRAPPER) FROM house;
                               json_query
------------------------------------------------------------------------
 [{"no": 2, "area": 80, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(1 row)
```
The result by jsonpath set-query operator:
```sql
SELECT js @*  '$.floor[*].apt[*] ? (@.area > 40 && @.area < 90)' FROM house;
             ?column?
-----------------------------------
 {"no": 2, "area": 80, "rooms": 3}
 {"no": 5, "area": 60, "rooms": 2}
(2 rows)
```

## SQL/JSON functions

###  Introduction to SQL/JSON functions

The SQL/JSON construction functions use values of SQL types and produce JSON values (JSON objects or JSON arrays) represented in SQL character or binary string types. They are mostly the same as corresponding json[b] construction functions.
- `JSON_OBJECT` -  construct a JSON\[b\] object
- `JSON_ARRAY` -  construct a JSON\[b\] array
- `JSON_ARRAYAGG` -  aggregates values as JSON\[b\] array
- `JSON_OBJECTAGG` - aggregates name/value pairs  as JSON\[b\] object

The SQL/JSON retrieval functions evaluate SQL/JSON path language expressions against JSON values, producing values of SQL/JSON types, which are converted to SQL types.  The SQL/JSON query functions all need a path specification, the JSON value to be input to that path specification for querying and processing, and optional parameter values passed to the path specification.

- `IS [NOT] JSON` - test whether a string value is a JSON text.
- `JSON_EXISTS` - determines whether a JSON value satisfies a search criterion.
- `JSON_VALUE` - extracts a scalar JSON value  and returns it as a native SQL type.
- `JSON_QUERY` - extracts a part  of JSON document and returns it as a JSON string.
- `JSON_TABLE` - query a JSON text and present the results as a relational table.


### Common syntax elements:
```
json_value_expression ::=
    expression [ FORMAT json_representation ]
json_representation ::=
	JSON [ ENCODING { UTF8 | UTF16 | UTF32 } ]
json_output_clause ::=
	RETURNING data_type [ FORMAT json_representation ]
json_api_common_syntax ::=
	json_value_expression , json_path_specification
	[ PASSING { json_value_expression AS identifier }[,...] ]
json_path_specification ::=
	character_string_literal | jsonpath_valued_expression /* extension */
```
Note: 
- Only `UTF8` encoding is supported
- `data_type` in `RETURNING` could be one of `json` (default), `jsonb`, `text`,  `bytea`.  PostgreSQL extends the supported types to records, arrays and domains.
- `json_representation` could be only `json`, so `FORMAT` is implemented only for the standard compatibility.
- The standard requires `character_string_literal` in `json_path_specification`,  PostgreSQL extends `json_path_specification` to a separate data type `jsonpath`.


### `JSON_OBJECT` - construct a JSON\[b\] object

Internally transformed into a `json[b]_build_object()` call depending on `RETURNING` type.
Syntax:
```
JSON_OBJECT (
  [ { expression {VALUE | ':'} json_value_expression }[,…] ]
  [ { NULL | ABSENT } ON NULL ]
  [ { WITH | WITHOUT } UNIQUE [ KEYS ] ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)

json_value_expression ::= expression [ FORMAT JSON ]
```

- `RETURNING` type:
-- `jsonb_build_object()` is used for `RETURNING jsonb`
-- `json_build_object()` - for other types
- Key uniqueness check: `{WITH|WITHOUT} UNIQUES [KEYS]`
- ability to omit keys with NULL values: `{ABSENT|NULL} ON NULL`

Examples:

```sql
SELECT JSON_OBJECT('a': 1, 'a': 2 WITH UNIQUE KEYS);
ERROR:  duplicate JSON key "a"

SELECT JSON_OBJECT('a': 1, 'a': 2);
    json_object     
--------------------
 {"a" : 1, "a" : 2}
(1 row)

SELECT JSON_OBJECT('a': 1, 'a': 2 RETURNING jsonb);
 json_object 
-------------
 {"a": 2}
(1 row)

-- Omitting keys with NULL values (keys  are not allowed to be NULL):
SELECT JSON_OBJECT('a': 1, 'b': NULL);
      json_object      
-----------------------
 {"a" : 1, "b" : null}
(1 row)

SELECT JSON_OBJECT('a': 1, 'b': NULL ABSENT ON NULL);
 json_object 
-------------
 {"a" : 1}
(1 row)

SELECT JSON_OBJECT('area' : 20 + 30, 'rooms': 2, 'no': 5) AS apt;
                 apt                  
--------------------------------------
 {"area" : 50, "rooms" : 2, "no" : 5}
(1 row)

-- by default json type is returned
SELECT JSON_OBJECT('area' : 20 + 30, 'rooms': 2, 'no': 5)->'no' AS no;
 no 
----
 5
(1 row)

-- jsonb: fields are ordered, duplicate fields removed
SELECT JSON_OBJECT('area' : 20 + 30, 'rooms': 2, 'no': 5, 'area' : NULL RETURNING jsonb) AS apt;
                 apt                 
-------------------------------------
 {"no": 5, "area": null, "rooms": 2}
(1 row)

-- WITH UNIQUE
SELECT JSON_OBJECT('area' : 20 + 30, 'rooms': 2, 'no': 5, 'area' : NULL WITH UNIQUE KEYS RETURNING jsonb) AS apt;
ERROR:  duplicate JSON key "area"

-- ABSENT ON NULL (last "area" key is skipped)
SELECT JSON_OBJECT('area' : 20 + 30, 'rooms': 2, 'no': 5, 'area' : NULL ABSENT ON NULL RETURNING jsonb) AS apt;
                apt                
-----------------------------------
 {"no": 5, "area": 50, "rooms": 2}
(1 row)

-- ABSENT ON NULL + WITH UNIQUE (key uniqueness is checked before NULLs are removed)
SELECT JSON_OBJECT('area' : 20 + 30, 'rooms': 2, 'no': 5, 'area' : NULL ABSENT ON NULL WITH UNIQUE KEYS RETURNING jsonb) AS apt;
ERROR:  duplicate JSON key "area"
```

### `JSON_OBJECTAGG` - aggregates name/value pairs  as JSON\[b\] object
`JSON_OBJECTAGG` is transformed into a `json[b]_object_agg()` call depending on `RETURNING` type.
Syntax:
```
JSON_OBJECTAGG (
  expression { VALUE | ':' } expression [ FORMAT JSON ]
  [ { NULL | ABSENT } ON NULL ]
  [ { WITH | WITHOUT } UNIQUE [ KEYS ] ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)
```
Options and `RETURNING` clause are the same as in `JSON_OBJECT`.

Examples:
```sql
SELECT JSON_OBJECTAGG('key' || i : 'val' ||  i)
FROM generate_series(1, 3) i;
                    json_objectagg                     
-------------------------------------------------------
 { "key1" : "val1", "key2" : "val2", "key3" : "val3" }
(1 row)

SELECT JSON_OBJECTAGG(k: v ABSENT ON NULL) AS apt
FROM (VALUES ('no', 5), ('area', 50), ('rooms', 2), ('foo', NULL)) kv(k, v);
                  apt
----------------------------------------
 { "no" : 5, "area" : 50, "rooms" : 2 }
(1 row)
```

### `JSON_ARRAY` - construct a JSON\[b\] array
Internally transformed into a `json[b]_build_array()` call depending on `RETURNING` type.
Syntax:
```
JSON_ARRAY (
  [ { expression [ FORMAT JSON ] }[, ...] ]
  [ { NULL | ABSENT } ON NULL ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)

JSON_ARRAY (
  query_expression
  [ RETURNING data_type [ FORMAT JSON ] ]
)SQL/JSON path expression finds
```
Options and `RETURNING` clause are the same as in `JSON_OBJECT`.
Note: `ON NULL` clause is not supported in subquery variant.

Examples:
```sql
SELECT JSON_ARRAY('string', 123, TRUE, ARRAY[1,2,3],
  '{"a": 1}'::jsonb, '[1, {"c": 3}]' FORMAT JSON
);
                       json_array                        
---------------------------------------------------------
 ["string", 123, true, [1,2,3], {"a": 1}, [1, {"c": 3}]]
(1 row)

SELECT JSON_ARRAY(SELECT * FROM generate_series(1, 3));
 json_array 
------------
 [1, 2, 3]
(1 row)

SELECT JSON_ARRAY(SELECT * FROM generate_series(1, 3) RETURNING bytea);
      json_array      
----------------------
 \x5b312c20322c20335d
(1 row)

SELECT JSON_ARRAY(
  json '{ "no": 1, "area": 40, "rooms": 1 }',
  JSON_OBJECT('no': 3, 'area': null, 'rooms': 2),
  JSON_OBJECT('no': 2, 'area': 80, 'rooms': 3 RETURNING text)
) AS apts;
                                                            apts                                                             
-----------------------------------------------------------------------------------------------------------------------------
 [{ "no": 1, "area": 40, "rooms": 1 }, {"no" : 3, "area" : null, "rooms" : 2}, "{\"no\" : 2, \"area\" : 80, \"rooms\" : 3}"]
(1 row)

```
### `JSON_ARRAYAGG` - aggregates values as JSON\[b\] array
Internally transformed into a `json[b]_agg()` call depending on `RETURNING` type.
Syntax:
```
JSON_ARRAYAGG (
  expression [ FORMAT JSON ]
  [ { NULL | ABSENT } ON NULL ]
  [ RETURNING data_type [ FORMAT JSON ] ]
)
```

Examples:
```sql
SELECT JSON_ARRAYAGG(i) FROM generate_series(1, 3) i;
 json_arrayagg 
---------------
 [1, 2, 3]
(1 row)

-- transform json data into relational form
CREATE TABLE house_apt AS
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    level int,
    NESTED PATH '$.apt[*]' COLUMNS (
      no int,
      area float,
      rooms int
    )
  )) jt;
  
SELECT * FROM house_apt;
 level | no |  area  | rooms 
-------+----+--------+-------
     1 |  1 |     40 |     1
     1 |  2 |     80 |     3
     1 |  3 | (null) |     2
     2 |  4 |    100 |     3
     2 |  5 |     60 |     2
(5 rows)

SELECT
  JSON_OBJECT(
    'level': level,
    'apt': JSON_ARRAYAGG(
      JSON_OBJECT(
        'no': no,
        'area': area,
        'rooms': rooms
      )
    )
  ) AS floor
FROM house_apt
GROUP BY level;
                                                              floor                                                                    
---------------------------------------------------------------------------------------------------------------------------------------------
 {"level" : 2, "apt" : [{"no" : 4, "area" : 100, "rooms" : 3}, {"no" : 5, "area" : 60, "rooms" : 2}]}
 {"level" : 1, "apt" : [{"no" : 1, "area" : 40, "rooms" : 1}, {"no" : 2, "area" : 80, "rooms" : 3}, {"no" : 3, "area" : null, "rooms" : 2}]}
(2 rows)

-- ABSENT ON NULL by default
SELECT JSON_ARRAYAGG(area) AS areas
FROM house_apt;
       areas       
-------------------
 [40, 80, 100, 60]
(1 row)

-- NULL ON NULL
SELECT JSON_ARRAYAGG(area NULL ON NULL) AS areas
FROM house_apt;
          areas          
-------------------------
 [40, 80, null, 100, 60]
(1 row)

-- record type serializing
SELECT JSON_ARRAYAGG(house_apt.*) AS floor
FROM house_apt
GROUP BY level;
                   floor                    
--------------------------------------------
 [{"level":2,"no":4,"area":100,"rooms":3}, +
  {"level":2,"no":5,"area":60,"rooms":2}]
 [{"level":1,"no":1,"area":40,"rooms":1},  +
  {"level":1,"no":2,"area":80,"rooms":3},  +
  {"level":1,"no":3,"area":null,"rooms":2}]
(2 rows)

-- nested JSON_ARRAYAGG
﻿SELECT
  JSON_ARRAYAGG(floor) AS floors
FROM (
  SELECT
    JSON_OBJECT(
      'level': level,
      'apt': JSON_ARRAYAGG(
        JSON_OBJECT(
          'no': no,
          'area': area,
          'rooms': rooms
        )
      )
    ) AS floor
  FROM house_apt
  GROUP BY level
) floor;
                                                                                                                       floors
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [{"level" : 2, "apt" : [{"no" : 4, "area" : 100, "rooms" : 3}, {"no" : 5, "area" : 60, "rooms" : 2}]}, {"level" : 1, "apt" : [{"no" : 1, "area" : 40, "rooms" : 1}, {"no" : 2, "area" : 80, "rooms" : 3}, {"no" : 3, "area" : null, "rooms" : 2}]}]
(1 row)

```

### `JSON_EXISTS` - determines whether a JSON value satisfies a search criterion

`JSON_EXISTS` is a predicate, that can be used to  test whether an SQL/JSON path expression returns one or more  SQL/JSON items.

Syntax:
```
JSON_EXISTS (
	json_api_common_syntax
	[ { TRUE | FALSE | UNKNOWN | ERROR } ON ERROR ]
)
```
Default is `FALSE ON ERROR`.

Examples:
```sql
-- analogous to jsonb '{"a": 1}' ? 'a'
SELECT JSON_EXISTS(jsonb '{"a": 1}', 'strict $.a');
 json_exists
-------------
 t
(1 row)
SELECT JSON_EXISTS(jsonb '{"a": [1,2,3]}', 'strict $.a[*] ? (@ > 2)');
 json_exists
-------------
 t
(1 row)
--  Using non-constant JSON path
SELECT JSON_EXISTS(jsonb '{"a": 123}', '$' || '.' || 'a');
 json_exists
-------------
 t
(1 row)
-- Error behavior (FALSE by default)
SELECT JSON_EXISTS(jsonb '{"a": [1,2,3]}', 'strict $.a[5]' ERROR ON ERROR);
ERROR: Invalid SQL/JSON subscript
SELECT JSON_EXISTS(jsonb '{"a": [1,2,3]}', 'lax $.a[5]' ERROR ON ERROR);
 json_exists
-------------
 f
(1 row)
```

###  <a name="json_value"></a>`JSON_VALUE` - extracts a scalar JSON value  and returns it as a native SQL type

Syntax:
```
JSON_VALUE (
  json_api_common_syntax
  [ RETURNING data_type ]
  [ { ERROR | NULL | DEFAULT expression } ON EMPTY ]
  [ { ERROR | NULL | DEFAULT expression } ON ERROR ]
)
```
The optional `RETURNING` clause performs a typecast. Without a `RETURNING` clause, `JSON_VALUE` returns a string of type `text`.

Examples:
```sql
-- Only singleton scalars are allowed here for returning:
SELECT JSON_VALUE(jsonb '[1]', 'strict $' ERROR ON ERROR);
ERROR: SQL/JSON scalar required
SELECT JSON_VALUE(jsonb '[1,2]', 'strict $[*]' ERROR ON ERROR);
ERROR: more than one SQL/JSON item

SELECT JSON_VALUE('"123.45"', '$' RETURNING float);
 json_value 
------------
     123.45
(1 row)

SELECT JSON_VALUE('"123.45"', '$' RETURNING int ERROR ON ERROR);
ERROR:  invalid input syntax for integer: "123.45"

SELECT JSON_VALUE('123.45', '$' RETURNING int);
 json_value 
------------
        123
(1 row)

SELECT JSON_VALUE('123.45', '$' RETURNING text);
 json_value 
------------
 123.45
(1 row)

SELECT JSON_VALUE('null', '$' RETURNING int ERROR ON ERROR);
 json_value 
------------
 (null)
(1 row)

SELECT JSON_VALUE('"03:04 2015-02-01"', '$.datetime("HH24:MI YYYY-MM-DD")' RETURNING date);
 json_value 
------------
 2015-02-01
(1 row)

SELECT JSON_VALUE('"03:04 2015-02-01"', '$' RETURNING date ERROR ON ERROR);
ERROR:  invalid input syntax for type date: "03:04 2015-02-01"
```

### `JSON_QUERY` - extracts a part  of JSON document and returns it as a JSON string

`JSON_QUERY` uses SQL/JSON path expression to extract a part\[s\] of JSON document. Since it should return a JSON string,  multiple parts should be wrapped   into an array  (`WRAPPER`) or the function should raise an exception.

Syntax:
```
JSON_QUERY (
	json_api_common_syntax
	[ RETURNING data_type [ FORMAT json_representation ] ]
	[ { WITHOUT | WITH { CONDITIONAL | [UNCONDITIONAL] } }
	[ ARRAY ] WRAPPER ]
	[ { KEEP | OMIT } QUOTES [ ON SCALAR STRING ] ]
	[ { ERROR | NULL | EMPTY { ARRAY | OBJECT } } ON EMPTY ]
	[ { ERROR | NULL | EMPTY { ARRAY | OBJECT } } ON ERROR ]
)
```
Default behavior of `WRAPPER` clause is not to wrap the matched  parts of JSON document  ( `WITHOUT WRAPPER` ) and to wrap unconditionally if wrap (`WITH UNCONDITIONAL WRAPPER`).

```sql
SELECT
    js,
    JSON_QUERY(js, 'lax $[*]') AS "without",
    JSON_QUERY(js, 'lax $[*]' WITH WRAPPER)  AS "with uncond",
    JSON_QUERY(js, 'lax $[*]' WITH CONDITIONAL WRAPPER) AS "with cond"
FROM
    (VALUES (jsonb '[]'), ('[1]'), ('[[1,2,3]]'),  ('[{"a": 1}]'), ('[1, null, "2"]')) foo(js);
       js       |  without  |  with uncond   |   with cond
----------------+-----------+----------------+----------------
 []             | (null)    | (null)         | (null)
 [1]            | 1         | [1]            | [1]
 [[1, 2, 3]]    | [1, 2, 3] | [[1, 2, 3]]    | [1, 2, 3]
 [{"a": 1}]     | {"a": 1}  | [{"a": 1}]     | {"a": 1}
 [1, null, "2"] | (null)    | [1, null, "2"] | [1, null, "2"]
(5 rows)
```
Quotes behavior (`KEEP` by default):
```sql
SELECT JSON_QUERY(jsonb '"aaa"', '$' RETURNING text);
 json_query 
------------
 "aaa"
(1 row)

SELECT JSON_QUERY(jsonb '"aaa"', '$' RETURNING text OMIT QUOTES);
 json_query
------------
 aaa
(1 row)
```

### `JSON_TABLE` - query a JSON text and present the results as a relational table

`JSON_TABLE` creates a relational view of JSON data.  It can be used only in `FROM` clause. 

`JSON_TABLE`  has several parameters:

1) The JSON value on which to operate.
2) An SQL/JSON path expression to specify zero or more rows. This  row pattern path expression is intended to produce an SQL/JSON sequence, with one SQL/JSON item for each row of the output table.
3) A `COLUMNS` clause to specify the schema of the output table. The `COLUMNS` can be nested.  The path expressions used in `COLUMNS` specification decompose SQL/JSON item on columns.
4) Optionally, `JSON_TABLE` can have `PLAN` clause, which specifies how to join nested columns.


`JSON_TABLE` internally uses `XMLTABLE` infrastructure (slightly modified).

Syntax:
```
JSON_TABLE (
json_api_common_syntax
COLUMNS ( json_table_column [, ...] )
[ PLAN ( json_table_plan ) |
  PLAN DEFAULT ( json_table_default_plan_choices ) ]
)

json_table_column ::=
    column_name FOR ORDINALITY

    | column_name data_type
		[ PATH json_path_specification ]
		/* behavior are the same as in JSON_VALUE */
		[ empty_behavior ON EMPTY ] 
		[ error_behavior ON ERROR ]

| column_name data_type FORMAT json_representation
		[ PATH json_path_specification ]
		/* behavior are the same as in JSON_QUERY */
	    [ wrapper_behavior WRAPPER ]
		[ quotes_behavior QUOTES [ ON SCALAR STRING ] ]
		[ empty_behavior ON EMPTY ]
		[ error_behavior ON ERROR ]
		
| NESTED PATH json_path_specification [ AS json_path_name ]
		COLUMNS ( json_table_column [, ...] )
json_table_default_plan_choices ::=
      { INNER | OUTER } [ , { CROSS | UNION } ]
	| { CROSS | UNION } [ , { INNER | OUTER } ]

json_table_plan ::=
	json_table_path_name
	| json_table_plan_parent/child
	| json_table_plan_sibling

json_table_plan_parent/child ::=
	  json_table_path_name OUTER json_table_plan_primary
	| json_table_path_name INNER json_table_plan_primary

json_table_plan_sibling ::=
	  json_table_plan_primary { UNION json_table_plan_primary }[...]
	| json_table_plan_primary { CROSS json_table_plan_primary }[...]

json_table_plan_primary ::=
		json_table_path_name | ( json_table_plan )
```

JSON_TABLE uses the row pattern  path expression to extract the parts of input document and decompose them to the columns using COLUMNS clause (schema) , optionaly using PLAN clause.  COLUMNS clause specifies path expressions for  each column, which evaluated on the extracted parts to produce columns values.  

The rows created by a  `JSON_TABLE`  are laterally joined, implicitly, to the row that generated them, so   there is no need to explicitly join a view produced by  JSON_TABLE  with the original table.

The `COLUMNS` clause can define several kinds of columns: ordinality columns, regular columns, formatted columns and nested columns.

In the following example, `JSON_TABLE` generates rows,  which obtained by matching the path expression `$.floor[*]` to input documents represented by `js`, laterally joined  to the table `house`.  Each generated row has columns defined by path expressions ( relative to the parent row path expression `$.floor[*]` ) in  `COLUMNS` clause:
 column  `level`  obtained by **implicit** path  `$.level`, column `num_apt`  and column `apts` explicitly defined by corresponding path expressions.  
```sql
-- basic example: regular and formatted columns, paths
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    level int,     -- PATH 'lax $.level' is used here by default
    num_apt int PATH '$.apt.size()', 
    apts jsonb FORMAT JSON PATH '$.apt'
  )) jt;
 level | num_apt |                                                    apts                                                     
-------+---------+-------------------------------------------------------------------------------------------------------------
     1 |       3 | [{"no": 1, "area": 40, "rooms": 1}, {"no": 2, "area": 80, "rooms": 3}, {"no": 3, "area": null, "rooms": 2}]
     2 |       2 | [{"no": 4, "area": 100, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}]
(2 rows)
```
An ordinality column provides a sequential numbering of rows. Row numbering is 1-based.

```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*].apt[*] ? (@.rooms > 1)' COLUMNS (
    id FOR ORDINALITY,
    no int,
    rooms int
  )) jt;
 id | no | rooms 
----+----+-------
  1 |  2 |     3
  2 |  3 |     2
  3 |  4 |     3
  4 |  5 |     2
(4 rows)
```
A regular column supports columns of scalar type. The column is produced using the semantics of `JSON_VALUE`, so it is possible to specify all  behavior clauses of `JSON_VALUE`, for example,  an optional `ON EMPTY` and `ON ERROR` clauses.

 The column has an **optional** path expression, called the column pattern, which can be defaulted from the column name. The column pattern is used to search for the column within the current SQL/JSON item produced by the row pattern. 
```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*].apt[*]' COLUMNS (
    no int,
    area float PATH '$.area / 100',
    area_type text PATH '$.area.type()'
  )) jt;
 no | area | area_type 
----+------+-----------
  1 |  0.4 | number
  2 |  0.8 | number
  3 |      | null
  4 |    1 | number
  5 |  0.6 | number
(5 rows)

-- regular datetime columns
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.info.dates[*]' COLUMNS (
    str text PATH '$',
    dt date PATH '$.datetime("DD-MM-YYYY")',
    ts timestamp PATH '$.datetime("DD-MM-YYYY HH24:MI:SS")',
    tstz timestamptz PATH '$.datetime("DD-MM-YYYY HH24:MI:SS TZH:TZM", "+03")'
  )) jt;
           str           |     dt     |         ts          |          tstz          
-------------------------+------------+---------------------+------------------------
 01-02-2015              | 2015-02-01 | 2015-02-01 00:00:00 | 2015-02-01 00:00:00+03
 04-10-1957 19:28:34 +00 | 1957-10-04 | 1957-10-04 19:28:34 | 1957-10-04 22:28:34+03
 12-04-1961 09:07:00 +03 | 1961-04-12 | 1961-04-12 09:07:00 | 1961-04-12 09:07:00+03
(3 rows)

-- regular columns: DEFAULT ON EMPTY behavior
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*].apt[*]' COLUMNS (
    area float4 PATH '$.area ? (@ != null)' DEFAULT 0 ON EMPTY    
  )) jt;
 area 
------
   40
   80
    0
  100
   60
(5 rows)

-- regular columns: ERROR ON EMPTY behavior
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*].apt[*]' COLUMNS (
    area float4 PATH '$.area ? (@ != null)' ERROR ON EMPTY ERROR ON ERROR
  )) jt;
ERROR:  no SQL/JSON item

-- regular columns: DEFAULT ON ERROR behavior
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*].apt[*]' COLUMNS (
    area text PATH '$.area * 100' DEFAULT 'Unknown' ON ERROR
  )) jt;
  area   
---------
 4000
 8000
 Unknown
 10000
 6000
(5 rows)

-- JSON_TABLE ON ERROR behavior: EMPTY ON ERROR is by default
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, 'strict $.foo[*]' COLUMNS (
    bar int
  )) jt;
 bar 
-----
(0 rows)

-- JSON_TABLE ON ERROR behavior: ERROR ON ERRROR
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, 'strict $.foo[*]' COLUMNS (
    bar int
  ) ERROR ON ERROR) jt;
ERROR:  SQL/JSON member not found
  
-- JSON_TABLE ON ERROR behavior overrides default ON ERROR behavior of its columns
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    bar int PATH 'strict $.bar' -- NULL ON ERROR is by default here
  ) ERROR ON ERROR) jt;
ERROR:  SQL/JSON member not found
```
Formatted columns are used for returning of composite SQL/JSON items, they internally transformed into `JSON_QUERY`,  so it is possible to specify all  behavior clauses of `JSON_QUERY`, for example,  an optional `ON EMPTY` and `ON ERROR` clauses.
```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    floor jsonb FORMAT JSON PATH '$'
  )) jt;
                                                              floor                                                               
----------------------------------------------------------------------------------------------------------------------------------
 {"apt": [{"no": 1, "area": 40, "rooms": 1}, {"no": 2, "area": 80, "rooms": 3}, {"no": 3, "area": null, "rooms": 2}], "level": 1}
 {"apt": [{"no": 4, "area": 100, "rooms": 3}, {"no": 5, "area": 60, "rooms": 2}], "level": 2}
(2 rows)

-- returning of composite JSON items without FORMAT JSON => error (NULL ON ERROR by default)
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    floor jsonb PATH '$'
  )) jt;
 floor
--------
 (null)
 (null)
(2 rows)

-- returning of composite JSON items without FORMAT JSON => error (ERROR ON ERROR)
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    floor jsonb PATH '$'
  ) ERROR ON ERROR) jt;
ERROR:  SQL/JSON scalar required
```
The nested `COLUMNS` clause begins with the keyword `NESTED`, followed by a path and an optional path name. The path provides a refined context for the nested columns. The primary use of the path name is if the user wishes to specify an explicit plan. After the prolog to specify the path and path name, there is a `COLUMNS` clause, which has the same capabilities already considered. The `NESTED` clause allows unnesting of (even deeply) nested JSON objects/arrays in one invocation rather than chaining several `JSON_TABLE` expressions in the SQL-statement.  

```sql
-- nested columns (1-level)
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    level int,
    NESTED PATH '$.apt[*]' COLUMNS (
      no int,
      area float,
      rooms int
    )
  )) jt;

 level | no |  area  | rooms 
-------+----+--------+-------
     1 |  1 |     40 |     1
     1 |  2 |     80 |     3
     1 |  3 | (null) |     2
     2 |  4 |    100 |     3
     2 |  5 |     60 |     2
(5 rows)
```
Every path may be followed by a path name using an `AS` clause. Path names are identifiers and must be unique and don't coincide with the column names. Path names are used in the `PLAN` clause to express the desired output plan. `PLAN` clause could be `INNER`, `OUTER`, `UNION` and `CROSS`, which correspond to `INNER JOIN`, `LEFT OUTER JOIN`, `FULL OUTER JOIN` and `CROSS JOIN` respectively.  If there is an explicit `PLAN` clause, all path names must be explicit and appear in the `PLAN` clause exactly once.

`INNER` and `OUTER`  (default) are used for parent/child relationship and it is mandatory to specify the first operand (path name in `AS` clause) and it must be an ancestor of all path names in the second operand. 

`UNION` expresses semantics of a `FULL OUTER JOIN` and is default with sibling relationship.


```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]'  AS lvl COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.area > 1000)' AS big COLUMNS (
      no int
    )
  ) PLAN (lvl OUTER big) ) jt;
 level |   no
-------+--------
     1 | (null)
     2 | (null)
(2 rows)

-- Default plan clause is  OUTER for parent/child relationship
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.area > 1000)' COLUMNS ( -- returns no items
      no int
    )
  )) jt;
 level |   no
-------+--------
     1 | (null)
     2 | (null)
(2 rows)
```
Example of nested columns (2-level).
```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$' COLUMNS (
    city text PATH '$.address.city',
    NESTED PATH '$.floor[*]' COLUMNS (
      level int,
      NESTED PATH '$.apt[*]' COLUMNS (
        no int,
        area float,
        rooms int
      )
    )
  )) jt;
  city  | level | no |  area  | rooms
--------+-------+----+--------+-------
 Moscow |     1 |  1 |     40 |     1
 Moscow |     1 |  2 |     80 |     3
 Moscow |     1 |  3 | (null) |     2
 Moscow |     2 |  4 |    100 |     3
 Moscow |     2 |  5 |     60 |     2
(5 rows)
```
Two nested paths on the same level are union-joined by default.
```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' COLUMNS (
    level int,
    NESTED PATH '$.apt[*]' COLUMNS (
      no1 int PATH '$.no'
    ),
    NESTED PATH '$.apt[*]' COLUMNS (
      no2 int PATH '$.no'
    )
  )) jt;
 level |  no1   |  no2
-------+--------+--------
     1 |      1 | (null)
     1 |      2 | (null)
     1 |      3 | (null)
     1 | (null) |      1
     1 | (null) |      2
     1 | (null) |      3
     2 |      4 | (null)
     2 |      5 | (null)
     2 | (null) |      4
     2 | (null) |      5
(10 rows)
```
 `PLAN DEFAULT (INNER)` is equivalent to explicit `PLAN` clause.
 
```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.area > 1000)' AS apt COLUMNS ( -- returns 0 items
      no int
    )
  ) PLAN DEFAULT (INNER)) jt;
 level | no
-------+----
(0 rows)

SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.area > 1000)' AS apt COLUMNS ( -- returns 0 items
      no int
    )
  ) PLAN (floor INNER apt)) jt;
 level | no
-------+----
(0 rows)
```
`PLAN DEFAULT (CROSS)` is equivalent to explicit `PLAN` clause for sibling columns.
```sql
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*]' AS apt1 COLUMNS (
      no1 int PATH '$.no'
    ),
    NESTED PATH '$.apt[*]' AS apt2 COLUMNS (
      no2 int PATH '$.no'
    )
  ) PLAN DEFAULT (CROSS)) jt;
 level | no1 | no2
-------+-----+-----
     1 |   1 |   1
     1 |   1 |   2
     1 |   1 |   3
     1 |   2 |   1
     1 |   2 |   2
     1 |   2 |   3
     1 |   3 |   1
     1 |   3 |   2
     1 |   3 |   3
     2 |   4 |   4
     2 |   4 |   5
     2 |   5 |   4
     2 |   5 |   5
(13 rows)
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*]' AS apt1 COLUMNS (
      no1 int PATH '$.no'
    ),
    NESTED PATH '$.apt[*]' AS apt2 COLUMNS (
      no2 int PATH '$.no'
    )
  ) PLAN (floor OUTER (apt1 CROSS apt2))) jt;
 level | no1 | no2
-------+-----+-----
     1 |   1 |   1
     1 |   1 |   2
     1 |   1 |   3
     1 |   2 |   1
     1 |   2 |   2
     1 |   2 |   3
     1 |   3 |   1
     1 |   3 |   2
     1 |   3 |   3
     2 |   4 |   4
     2 |   4 |   5
     2 |   5 |   4
     2 |   5 |   5
(13 rows)
```
Combination of `OUTER`/`INNER` and `UNION`/`CROSS` joins:
```sql
-- OUTER, UNION is the same as INNER, UNION
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.no > 3)' AS apt1 COLUMNS ( no1 int PATH '$.no' ),
    NESTED PATH '$.apt[*]'              AS apt2 COLUMNS ( no2 int PATH '$.no' )
  ) PLAN DEFAULT (OUTER, UNION)) jt;
 level |  no1   |  no2
-------+--------+--------
     1 | (null) |      1
     1 | (null) |      2
     1 | (null) |      3
     2 |      4 | (null)
     2 |      5 | (null)
     2 | (null) |      4
     2 | (null) |      5
(7 rows)

-- INNER, UNION
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.no > 3)' AS apt1 COLUMNS ( no1 int PATH '$.no' ),
    NESTED PATH '$.apt[*]'              AS apt2 COLUMNS ( no2 int PATH '$.no' )
  ) PLAN DEFAULT (INNER, UNION)) jt;
 level |  no1   |  no2
-------+--------+--------
     1 | (null) |      1
     1 | (null) |      2
     1 | (null) |      3
     2 |      4 | (null)
     2 |      5 | (null)
     2 | (null) |      4
     2 | (null) |      5
(7 rows)

-- OUTER, CROSS
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.no > 3)' AS apt1 COLUMNS ( no1 int PATH '$.no' ),
    NESTED PATH '$.apt[*]'              AS apt2 COLUMNS ( no2 int PATH '$.no' )
  ) PLAN DEFAULT (OUTER, CROSS)) jt;
 level |  no1   |  no2
-------+--------+--------
     1 | (null) | (null)
     2 |      4 |      4
     2 |      4 |      5
     2 |      5 |      4
     2 |      5 |      5
(5 rows)

-- INNER, CROSS
SELECT
  jt.*
FROM
  house,
  JSON_TABLE(js, '$.floor[*]' AS floor COLUMNS (
    level int,
    NESTED PATH '$.apt[*] ? (@.no > 3)' AS apt1 COLUMNS ( no1 int PATH '$.no' ),
    NESTED PATH '$.apt[*]'              AS apt2 COLUMNS ( no2 int PATH '$.no' )
  ) PLAN DEFAULT (INNER, CROSS)) jt;
 level | no1 | no2
-------+-----+-----
     2 |   4 |   4
     2 |   4 |   5
     2 |   5 |   4
     2 |   5 |   5
(4 rows)

-- OUTER, UNION
SELECT
  jt.*
FROM
  JSON_TABLE('[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]', '$[*]' AS root
    COLUMNS (
      n int,
      NESTED PATH '$.a[*] ? (@ > 2)' AS ap1 COLUMNS ( a1 int PATH '$' ),
      NESTED PATH '$.a[*]'           AS ap2 COLUMNS ( a2 int PATH '$' )
    ) PLAN DEFAULT (OUTER, UNION)
  ) jt;
 n |   a1   |   a2
---+--------+--------
 1 | (null) |      1
 1 | (null) |      2
 2 |      3 | (null)
 2 |      4 | (null)
 2 | (null) |      3
 2 | (null) |      4
 3 | (null) | (null)
(7 rows)

-- INNER, UNION
SELECT
  jt.*
FROM
  JSON_TABLE('[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]', '$[*]' AS root
    COLUMNS (
      n int,
      NESTED PATH '$.a[*] ? (@ > 2)' AS ap1 COLUMNS ( a1 int PATH '$' ),
      NESTED PATH '$.a[*]'           AS ap2 COLUMNS ( a2 int PATH '$' )
    ) PLAN DEFAULT (INNER, UNION)
  ) jt;
 n |   a1   |   a2
---+--------+--------
 1 | (null) |      1
 1 | (null) |      2
 2 |      3 | (null)
 2 |      4 | (null)
 2 | (null) |      3
 2 | (null) |      4
(6 rows)

--OUTER, CROSS
SELECT
  jt.*
FROM
  JSON_TABLE('[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]', '$[*]' AS root
    COLUMNS (
      n int,
      NESTED PATH '$.a[*] ? (@ > 2)' AS ap1 COLUMNS ( a1 int PATH '$' ),
      NESTED PATH '$.a[*]'           AS ap2 COLUMNS ( a2 int PATH '$' )
    ) PLAN DEFAULT (OUTER, CROSS)
  ) jt;
 n |   a1   |   a2
---+--------+--------
 1 | (null) | (null)
 2 |      3 |      3
 2 |      3 |      4
 2 |      4 |      3
 2 |      4 |      4
 3 | (null) | (null)
(6 rows)

-- INNER, CROSS
SELECT
  jt.*
FROM
  JSON_TABLE('[{"a": [1, 2], "n": 1}, {"a": [3, 4], "n": 2}, {"a": [], "n": 3}]', '$[*]' AS root
    COLUMNS (
      n int,
      NESTED PATH '$.a[*] ? (@ > 2)' AS ap1 COLUMNS ( a1 int PATH '$' ),
      NESTED PATH '$.a[*]'           AS ap2 COLUMNS ( a2 int PATH '$' )
    ) PLAN DEFAULT (INNER, CROSS)
  ) jt;
 n | a1 | a2
---+----+----
 2 |  3 |  3
 2 |  3 |  4
 2 |  4 |  3
 2 |  4 |  4
(4 rows)
```

## SQL/JSON conformance

- `like_regex` supports posix regular expressions,  while the  standard requires xquery regexps.
-  Not supported (due to unresolved conflicts in SQL grammar):
   - `expr FORMAT JSON IS [NOT] JSON`
   - `JSON_OBJECT(KEY key VALUE value, ...)`
   - `JSON_ARRAY(SELECT ... FORMAT JSON ...)`
   - `JSON_ARRAY(SELECT ... (ABSENT|NULL) ON NULL ...)`
   - `json_path_specification` extended to be an expression of `jsonpath` type. The standard requires  it `character_string_literal`.
-  Use boolean  expression on the path, PostgreSQL extension
-  Default timezone added to `datetime()` as second argument. That helped to keep `jsonpath` operators and functions to be immutable.
- `.**`  - recursive wildcard member accessor, PostgreSQL extension
- `json[b] op jsonpath` - PostgreSQL extension
- `[path]` - wrap SQL/JSON sequences into an array - PostgreSQL extension


 ## Links
* Github Postgres Professional repository
https://github.com/postgrespro/sqljson
*  WEB-interface to play with SQL/JSON
http://sqlfiddle.postgrespro.ru/#!21/0/2580
* Try SQL/JSON on your android phone 
https://obartunov.livejournal.com/199005.html
*  SQL/JSON standard-2016 conformance for PostgreSQL, Oracle, SQL Server and MySQL
 https://obartunov.livejournal.com/200076.html
* Technical Report (SQL/JSON) - available for free
http://standards.iso.org/i/PubliclyAvailableStandards/c067367_ISO_IEC_TR_19075-6_2017.zip
* Jsonb roadmap - talk at PGConf.eu, 2018
http://www.sai.msu.su/~megera/postgres/talks/sqljson-pgconf.eu-2017.pdf

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc1NjcxNDgyNl19
-->
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTgwMjQzOTRdfQ==
-->
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc1MzY5NjI5Ml19
-->
<!--stackedit_data:
eyJoaXN0b3J5IjpbOTY3OTAyOTUwLC0xMzI5OTAxNTY5XX0=
-->