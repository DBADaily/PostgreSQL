# PostgreSQL hstore

# What is hstore

hstore is a PostgreSQL built-in extension. 

With hstore extension, you can create a table with `hstore` data type columns for storing sets of key value pairs within a single PostgreSQL value. Keys and  values are simply text strings.

# Scenarios

hstore can be useful in various scenarios, such as:

1. columns will have different attributes depending values of other columns

2. rows with many attributes that are rarely examined

3. semi-structured data

Now let's see how to make full use of hstore step by step.

# Create hstore extension

Try to create hstore extension with a normal user.

```
$ psql -d alvindb -U alvin
alvindb=> CREATE EXTENSION hstore;
ERROR:  permission denied to create extension "hstore"
HINT:  Must be superuser to create this extension.
```

As you can see, superuser is needed to create this extension.

Now create hstore extension with super user postgres.

```
$ psql -d alvindb
alvindb=# CREATE EXTENSION hstore;
CREATE EXTENSION
alvindb=# \dx hstore
                         List of installed extensions
  Name  | Version | Schema |                   Description                    
--------+---------+--------+--------------------------------------------------
 hstore | 1.5     | public | data type for storing sets of (key, value) pairs
(1 row)
```

# Create a table with a hstore column

Create a table with `hstore` data type.

Table creation process.

```
$ psql -d alvindb -U alvin
alvindb=> CREATE TABLE person (
    person_id   BIGSERIAL PRIMARY KEY,
    person_name VARCHAR(64),
    hobbies     hstore,
    create_time TIMESTAMP DEFAULT now() NOT NULL,
    update_time TIMESTAMP DEFAULT now() NOT NULL
);
CREATE TABLE
alvindb=> COMMENT ON TABLE person IS 'person table';
COMMENT
alvindb=> COMMENT ON COLUMN person.person_id IS 'person unique identifier';
COMMENT
alvindb=> COMMENT ON COLUMN person.person_name IS 'person name';
COMMENT
alvindb=> COMMENT ON COLUMN person.hobbies IS 'hobbies with levels';
COMMENT
alvindb=> COMMENT ON COLUMN person.create_time IS 'timestamp created';
COMMENT
alvindb=> COMMENT ON COLUMN person.update_time IS 'timestamp updated';
COMMENT
alvindb=> CREATE INDEX ON person (person_name);
CREATE INDEX
```

Check table structure.

```
alvindb=> \dt+ person
                   List of relations
 Schema |  Name  | Type  | Owner | Size  | Description  
--------+--------+-------+-------+-------+--------------
 alvin  | person | table | alvin | 32 kB | person table
(1 row)
alvindb=> \d+ person
                                                                       Table "alvin.person"
   Column    |            Type             | Collation | Nullable |                  Default                  | Storage  | Stats target |       Description        
-------------+-----------------------------+-----------+----------+-------------------------------------------+----------+--------------+--------------------------
 person_id   | bigint                      |           | not null | nextval('person_person_id_seq'::regclass) | plain    |              | person unique identifier
 person_name | character varying(64)       |           |          |                                           | extended |              | person name
 hobbies     | hstore                      |           |          |                                           | extended |              | hobbies with levels
 create_time | timestamp without time zone |           | not null | now()                                     | plain    |              | timestamp created
 update_time | timestamp without time zone |           | not null | now()                                     | plain    |              | timestamp updated
Indexes:
    "person_pkey" PRIMARY KEY, btree (person_id)
    "person_person_name_idx" btree (person_name)
```

# hstore data manipulations

## Insert data

hstore data can be inserted in various ways as follows.

```
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Alvin', 'English=>B2');
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Tom', '"Dota 2"=>"Level 66"');
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Lily', '"Piano"=>"Grade 6","Guitar"=>"Level 6"');
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('David', '"Go"=>"Grade 6","Chess"=>"Master"'::hstore);
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Lucy', hstore('Ballet','Level 3'));
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Kate', hstore(ARRAY['Ballet','Level 2','Piano','Grade 5']));
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Julia', hstore(ARRAY[['Ballet','Level 5'],['Piano','Grade 6']]));
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Rose', hstore(ARRAY['Ballet','Piano'],ARRAY['Level 1','Grade 2']));
INSERT 0 1
alvindb=> INSERT INTO person(person_name,hobbies) VALUES ('Robert', hstore(row('Guitar','Piano','Chess')));
INSERT 0 1
alvindb=> SELECT person_name,hobbies FROM person ORDER BY person_id;
 person_name |                   hobbies                    
-------------+----------------------------------------------
 Alvin       | "English"=>"B2"
 Tom         | "Dota 2"=>"Level 66"
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
 David       | "Go"=>"Grade 6", "Chess"=>"Master"
 Lucy        | "Ballet"=>"Level 3"
 Kate        | "Piano"=>"Grade 5", "Ballet"=>"Level 2"
 Julia       | "Piano"=>"Grade 6", "Ballet"=>"Level 5"
 Rose        | "Piano"=>"Grade 2", "Ballet"=>"Level 1"
 Robert      | "f1"=>"Guitar", "f2"=>"Piano", "f3"=>"Chess"
(9 rows)
```

## Add key value pairs

Here is how to append key value pairs to a hstore column.

```
alvindb=> UPDATE person SET hobbies = coalesce(hobbies, '') || '"Math"=>"State Level","Computer"=>"Level 4"' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
(1 row)
```

You might have noticed `coalesce` in `coalesce(hobbies, '')` which is to avoid the **`NULL` concatenation trap** that will produce unexpected results.

Let's first set the hstore column to `NULL`, then test concatenation of hstore and `NULL`.

```
alvindb=> UPDATE person SET hobbies = NULL WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> UPDATE person SET hobbies = hobbies || '"Math"=>"State Level","Computer"=>"Level 4"' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name | hobbies 
-------------+---------
 Alvin       | 
(1 row)
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin' AND hobbies IS NULL;
 person_name | hobbies 
-------------+---------
 Alvin       | 
(1 row)
```

The result is `NULL` since when you concatenate anything with `NULL`, the result will always be `NULL`.

How about empty string `''` ? Does it have something like the **`NULL` concatenation trap**? As you see from following test, the answer is negative.

```
alvindb=> UPDATE person SET hobbies = '' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name | hobbies 
-------------+---------
 Alvin       | 
(1 row)
alvindb=> UPDATE person SET hobbies = hobbies || '"Math"=>"State Level","Computer"=>"Level 4"' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                   hobbies                    
-------------+----------------------------------------------
 Alvin       | "Math"=>"State Level", "Computer"=>"Level 4"
(1 row)
```

## Delete key value pairs

### Delete pair with matching key

Delete one pair with matching key

```
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
(1 row)
alvindb=> UPDATE person SET hobbies = hobbies - 'English'::text WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                   hobbies                    
-------------+----------------------------------------------
 Alvin       | "Math"=>"State Level", "Computer"=>"Level 4"
(1 row)
```

Delete two pairs with matching key

```
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
(1 row)
alvindb=> UPDATE person SET hobbies = hobbies - 'English'::text - 'Math'::text WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |        hobbies        
-------------+-----------------------
 Alvin       | "Computer"=>"Level 4"
(1 row)
```

### Delete pairs with matching keys

```
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
(1 row)
alvindb=> UPDATE person SET hobbies = hobbies - ARRAY['English','Math'] WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |        hobbies        
-------------+-----------------------
 Alvin       | "Computer"=>"Level 4"
(1 row)
```

### Delete pairs with matching hstore

```
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
(1 row)
alvindb=> UPDATE person SET hobbies = hobbies - '"English"=>"B2","Computer"=>"Level 4"' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |        hobbies        
-------------+-----------------------
 Alvin       | "Math"=>"State Level"
(1 row)
```

For unmatching hstore, it will not delete.

```
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
(1 row)
alvindb=> UPDATE person SET hobbies = hobbies - '"English"=>"B2","Computer"=>"Level 5"' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                   hobbies                    
-------------+----------------------------------------------
 Alvin       | "Math"=>"State Level", "Computer"=>"Level 4"
(1 row)
```

After all pairs are deleted, the final hstore value will be an empty string `''` instead of `NULL`.

```
alvindb=> UPDATE person SET hobbies = hobbies - '"English"=>"B2","Math"=>"State Level","Computer"=>"Level 4"' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name | hobbies 
-------------+---------
 Alvin       | 
(1 row)
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin' AND hobbies IS NULL;
 person_name | hobbies 
-------------+---------
(0 rows)
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin' AND hobbies = '';
 person_name | hobbies 
-------------+---------
 Alvin       | 
(1 row)
```

# Get hstore

Following examples will be based on data below.

```
alvindb=> UPDATE person SET hobbies = coalesce(hobbies, '') || '"English"=>"B2","Math"=>"State Level","Computer"=>"Level 4"' WHERE person_name = 'Alvin';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE person_name = 'Alvin';
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
(1 row)
alvindb=> SELECT person_name,hobbies FROM person ORDER BY person_id;
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
 Tom         | "Dota 2"=>"Level 66"
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
 David       | "Go"=>"Grade 6", "Chess"=>"Master"
 Lucy        | "Ballet"=>"Level 3"
 Kate        | "Piano"=>"Grade 5", "Ballet"=>"Level 2"
 Julia       | "Piano"=>"Grade 6", "Ballet"=>"Level 5"
 Rose        | "Piano"=>"Grade 2", "Ballet"=>"Level 1"
 Robert      | "f1"=>"Guitar", "f2"=>"Piano", "f3"=>"Chess"
(9 rows)
```

## Get hstore keys

### Get hstore keys as an ARRAY

```
alvindb=> SELECT akeys(hobbies) FROM person WHERE person_name = 'Alvin';
          akeys          
-------------------------
 {Math,English,Computer}
(1 row)
```

### Get hstore keys as a set

With skeys function

```
alvindb=> SELECT skeys(hobbies) FROM person WHERE person_name = 'Alvin' ORDER BY 1;
  skeys   
----------
 Computer
 English
 Math
(3 rows)
```

With each function

```
alvindb=> SELECT (each(hobbies)).key FROM person WHERE person_name = 'Alvin' ORDER BY 1;
   key    
----------
 Computer
 English
 Math
(3 rows)
```

## Get hstore values

### Get an hstore value by key

```
alvindb=> SELECT hobbies -> 'Math' AS math FROM person WHERE person_name = 'Alvin';
    math     
-------------
 State Level
(1 row)
```

### Get hstore values as an ARRAY

```
alvindb=> SELECT avals(hobbies) FROM person WHERE person_name = 'Alvin';
            avals             
------------------------------
 {"State Level",B2,"Level 4"}
(1 row)
```

### Get hstore values as a set

With svals function

```
alvindb=> SELECT svals(hobbies) FROM person WHERE person_name = 'Alvin' ORDER BY 1;
    svals    
-------------
 B2
 Level 4
 State Level
(3 rows)
```

With each function

```
alvindb=> SELECT (each(hobbies)).value FROM person WHERE person_name = 'Alvin' ORDER BY 1;
    value    
-------------
 B2
 Level 4
 State Level
(3 rows)
```

## Get hstore keys and values as a set

With each function

```
alvindb=> SELECT person_name,each(hobbies) FROM person WHERE person_name = 'Alvin';
 person_name |         each         
-------------+----------------------
 Alvin       | (Math,"State Level")
 Alvin       | (English,B2)
 Alvin       | (Computer,"Level 4")
(3 rows)
alvindb=> SELECT person_name,(each(hobbies)).* FROM person WHERE person_name = 'Alvin';
 person_name |   key    |    value    
-------------+----------+-------------
 Alvin       | Math     | State Level
 Alvin       | English  | B2
 Alvin       | Computer | Level 4
(3 rows)
alvindb=> SELECT person_name,(each(hobbies)).key,(each(hobbies)).value FROM person WHERE person_name = 'Alvin';
 person_name |   key    |    value    
-------------+----------+-------------
 Alvin       | Math     | State Level
 Alvin       | English  | B2
 Alvin       | Computer | Level 4
(3 rows)
```

Get all persons' hobbies.

```
alvindb=> SELECT person_name,(each(hobbies)).key,(each(hobbies)).value FROM person ORDER BY person_id;
 person_name |   key    |    value    
-------------+----------+-------------
 Alvin       | Math     | State Level
 Alvin       | English  | B2
 Alvin       | Computer | Level 4
 Tom         | Dota 2   | Level 66
 Lily        | Piano    | Grade 6
 Lily        | Guitar   | Level 6
 David       | Go       | Grade 6
 David       | Chess    | Master
 Lucy        | Ballet   | Level 3
 Lucy        | Guitar   | Level 6
 Kate        | Piano    | Grade 5
 Kate        | Ballet   | Level 2
 Julia       | Piano    | Grade 6
 Julia       | Ballet   | Level 5
 Rose        | Piano    | Grade 2
 Rose        | Ballet   | Level 1
 Robert      | f1       | Guitar
 Robert      | f2       | Piano
 Robert      | f3       | Chess
(19 rows)
```

Group hobbies by key.

```
alvindb=> SELECT (each(hobbies)).key,count(1) FROM person GROUP BY key ORDER BY count(1) DESC,key;
   key    | count 
----------+-------
 Ballet   |     4
 Piano    |     4
 Guitar   |     2
 Chess    |     1
 Computer |     1
 Dota 2   |     1
 English  |     1
 f1       |     1
 f2       |     1
 f3       |     1
 Go       |     1
 Math     |     1
(12 rows)
```

## Get a subset of an hstore

```
alvindb=> SELECT slice(hobbies,ARRAY['English','Chess','Math']) FROM person WHERE person_name = 'Alvin';
                 slice                  
----------------------------------------
 "Math"=>"State Level", "English"=>"B2"
(1 row)
alvindb=> SELECT (each(slice(hobbies,ARRAY['English','Chess','Math']))).* FROM person WHERE person_name = 'Alvin' ORDER BY 1;
   key   |    value    
---------+-------------
 English | B2
 Math    | State Level
(2 rows)
```

## Get hstore by containment

### Get hstore by containing key

```
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies ? 'Guitar' ORDER BY person_id;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
(1 row)
```

### Get hstore by containing all keys

```
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies ?& ARRAY['Piano','Ballet'] ORDER BY person_id;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Kate        | "Piano"=>"Grade 5", "Ballet"=>"Level 2"
 Julia       | "Piano"=>"Grade 6", "Ballet"=>"Level 5"
 Rose        | "Piano"=>"Grade 2", "Ballet"=>"Level 1"
(3 rows)
```

### Get hstore by containing any key

```
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies ?| ARRAY['Piano','Ballet'] ORDER BY person_id;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
 Lucy        | "Ballet"=>"Level 3"
 Kate        | "Piano"=>"Grade 5", "Ballet"=>"Level 2"
 Julia       | "Piano"=>"Grade 6", "Ballet"=>"Level 5"
 Rose        | "Piano"=>"Grade 2", "Ballet"=>"Level 1"
(5 rows)
```

### Get hstore by containing only non-NULL values

```
alvindb=> UPDATE person SET hobbies = coalesce(hobbies) || '"Guitar"=>NULL' WHERE person_name = 'Lucy';
UPDATE 1
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies ? 'Guitar' ORDER BY person_id;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
 Lucy        | "Ballet"=>"Level 3", "Guitar"=>NULL
(2 rows)
alvindb=> SELECT person_name,hobbies FROM person WHERE defined(hobbies,'Guitar') ORDER BY person_id;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
(1 row)
```

### Get hstore by containing right hstore

```
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies @> '"Ballet"=>"Level 2"' ORDER BY person_id;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Kate        | "Piano"=>"Grade 5", "Ballet"=>"Level 2"
(1 row)
```

### Get hstore by containing left hstore

```
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies <@ '"Ballet"=>"Level 5","Guitar"=>"Level 6","Piano"=>"Grade 6"' ORDER BY person_id;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
 Julia       | "Piano"=>"Grade 6", "Ballet"=>"Level 5"
(2 rows)
```

### Get hstore by containing same hstore pairs

```
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies = '"Piano"=>"Grade 2", "Ballet"=>"Level 1"' ORDER BY 1;
 person_name |                 hobbies                 
-------------+-----------------------------------------
 Rose        | "Piano"=>"Grade 2", "Ballet"=>"Level 1"
(1 row)
```

### Get hstore by containing all but same hstore pairs

```
alvindb=> SELECT person_name,hobbies FROM person WHERE hobbies <> '"Piano"=>"Grade 2", "Ballet"=>"Level 1"' ORDER BY 1;
 person_name |                            hobbies                            
-------------+---------------------------------------------------------------
 Alvin       | "Math"=>"State Level", "English"=>"B2", "Computer"=>"Level 4"
 David       | "Go"=>"Grade 6", "Chess"=>"Master"
 Julia       | "Piano"=>"Grade 6", "Ballet"=>"Level 5"
 Kate        | "Piano"=>"Grade 5", "Ballet"=>"Level 2"
 Lily        | "Piano"=>"Grade 6", "Guitar"=>"Level 6"
 Lucy        | "Ballet"=>"Level 3", "Guitar"=>NULL
 Robert      | "f1"=>"Guitar", "f2"=>"Piano", "f3"=>"Chess"
 Tom         | "Dota 2"=>"Level 66"
(8 rows)
```

# Construct hstore pairs with row or records

## Construct an hstore from a row

```
alvindb=> SELECT hstore(ROW(1,2,3));
             hstore              
---------------------------------
 "f1"=>"1", "f2"=>"2", "f3"=>"3"
(1 row)

alvindb=> SELECT hstore(ROW('Ballet','Guitar','Piano'));
                    hstore                     
-----------------------------------------------
 "f1"=>"Ballet", "f2"=>"Guitar", "f3"=>"Piano"
(1 row)
```

## Construct an hstore from records

Construct an hstore from a sub query.

```
alvindb=> SELECT hstore(h) FROM (SELECT person_id,person_name FROM person ORDER BY person_id) AS h;
                  hstore                   
-------------------------------------------
 "person_id"=>"1", "person_name"=>"Alvin"
 "person_id"=>"2", "person_name"=>"Tom"
 "person_id"=>"3", "person_name"=>"Lily"
 "person_id"=>"4", "person_name"=>"David"
 "person_id"=>"5", "person_name"=>"Lucy"
 "person_id"=>"6", "person_name"=>"Kate"
 "person_id"=>"7", "person_name"=>"Julia"
 "person_id"=>"8", "person_name"=>"Rose"
 "person_id"=>"9", "person_name"=>"Robert"
(9 rows)
```

Construct an hstore from a table query.

```
CREATE TABLE test_hstore (
    person_id   BIGSERIAL PRIMARY KEY,
    person_name VARCHAR(64)
);
INSERT INTO test_hstore(person_name) VALUES ('Alvin');
INSERT INTO test_hstore(person_name) VALUES ('Tom');
INSERT INTO test_hstore(person_name) VALUES ('Lily');
```

```
alvindb=> SELECT hstore(h) FROM test_hstore AS h ORDER BY person_id;
                  hstore                  
------------------------------------------
 "person_id"=>"1", "person_name"=>"Alvin"
 "person_id"=>"2", "person_name"=>"Tom"
 "person_id"=>"3", "person_name"=>"Lily"
(3 rows)
```

# Convert hstore

## hstore to ARRAY

```
alvindb=> SELECT hstore_to_ARRAY(hobbies) FROM person WHERE person_name = 'Alvin';
                  hstore_to_ARRAY                   
----------------------------------------------------
 {Math,"State Level",English,B2,Computer,"Level 4"}
(1 row)
```

## hstore to matrix ARRAY

```
alvindb=> SELECT hstore_to_matrix(hobbies) FROM person WHERE person_name = 'Alvin';
                     hstore_to_matrix                     
----------------------------------------------------------
 {{Math,"State Level"},{English,B2},{Computer,"Level 4"}}
```

## hstore to jsonb

```
alvindb=> SELECT hstore_to_jsonb(hobbies) FROM person WHERE person_name = 'Alvin';
                         hstore_to_jsonb                         
-----------------------------------------------------------------
 {"Math": "State Level", "English": "B2", "Computer": "Level 4"}
(1 row)
```

# hstore operator function map

In fact, all operators are implemented with functions.

All above hstore operators can be replaced with corresponding hstore functions.

While operators can be helpful and handy, operator functions are self explanatory.

```
alvindb=> SELECT
    lpad(replace(o.oprleft::regtype::text, '-', ''), 10, ' ') 
    || ' ' 
    || rpad(o.oprname, 4, ' ') 
    || ' ' 
    || rpad(o.oprright::regtype::text, 6, ' ') 
    || ' = ' 
    || o.oprresult::regtype::text AS expr,
    o.oprcode::regproc AS operator_function
FROM
    pg_catalog.pg_depend    d,
    pg_catalog.pg_extension e,
    pg_catalog.pg_class c,
    pg_type t,
    pg_operator o
WHERE
        e.extname = 'hstore'
    AND d.refclassid = pg_catalog.regclass('pg_catalog.pg_extension')
    AND d.refobjid = e.oid
    AND d.deptype = 'e'
    and d.classid = c.oid
    and c.reltype = t.oid
    and t.typname = 'pg_operator'
    and d.objid = o.oid
ORDER BY
    o.oprname DESC,
    o.oprleft,
    o.oprright,
    o.oprresult;
                expr                 | operator_function 
-------------------------------------+-------------------
     hstore ~    hstore = boolean    | hs_contained
     hstore ||   hstore = hstore     | hs_concat
     hstore @>   hstore = boolean    | hs_contains
     hstore @    hstore = boolean    | hs_contains
     hstore ?|   text[] = boolean    | exists_any
     hstore ?&   text[] = boolean    | exists_all
     hstore ?    text   = boolean    | exist
     hstore =    hstore = boolean    | hstore_eq
     hstore <@   hstore = boolean    | hs_contained
     hstore <>   hstore = boolean    | hstore_ne
     hstore ->   text   = text       | fetchval
     hstore ->   text[] = text[]     | slice_ARRAY
     hstore -    text   = hstore     | public.delete
     hstore -    text[] = hstore     | public.delete
     hstore -    hstore = hstore     | public.delete
            %%   hstore = text[]     | hstore_to_ARRAY
            %#   hstore = text[]     | hstore_to_matrix
     hstore #>=# hstore = boolean    | hstore_ge
     hstore #>#  hstore = boolean    | hstore_gt
 anyelement #=   hstore = anyelement | populate_record
     hstore #<=# hstore = boolean    | hstore_le
     hstore #<#  hstore = boolean    | hstore_lt
(22 rows)
```

## Deprecated operators

Prior to PostgreSQL 8.2, the containment operators `@>` and `<@` were called `@` and `~`, respectively. These names are still available, but are deprecated and will eventually be removed. Notice that the old names are reversed from  the convention formerly followed by the core geometric data types.