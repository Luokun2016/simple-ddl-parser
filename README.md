## Simple DDL Parser

![badge1](https://img.shields.io/pypi/v/simple-ddl-parser) ![badge2](https://img.shields.io/pypi/l/simple-ddl-parser) ![badge3](https://img.shields.io/pypi/pyversions/simple-ddl-parser) ![workflow](https://github.com/xnuinside/simple-ddl-parser/actions/workflows/main.yml/badge.svg)

Build with ply (lex & yacc in python). A lot of samples in 'tests/.

### Is it Stable?

Yes, library already has about 4000 usage per day, you can check statistics by yourself - https://pypistats.org/packages/simple-ddl-parser.

As maintainer I guarantee that any backward incompatible changes will not be done in patch or minor version. Only additionals & new features.

However, in process of adding support for new statements & features I see that output can be structured more optimal way and I hope to release version `1.0.*` with more struct output result. But, it will not be soon, first of all, I want to add support for so much statements as I can. So I don't think make sense to expect version 1.0.* before, for example, version `0.26.0` :)

### How does it work?

Parser tested on different DDLs for PostgreSQL & Hive. But idea to support as much as possible DDL dialects (Vertica, Oracle, Hive, MsSQL, etc.). You can check dialects sections after `Supported Statements` section to get more information that statements from dialects already supported by parser.
**If you need some statement, that not supported by parser yet**: please provide DDL example & information about that is it SQL dialect or DB.

Types that are used in your DB does not matter, so parser must also work successfuly to any DDL for SQL DB. Parser is NOT case sensitive, it did not expect that all queries will be in upper case or lower case. So you can write statements like this:

```sql

    Alter Table Persons ADD CONSTRAINT CHK_PersonAge CHECK (Age>=18 AND City='Sandnes');

```

It will be parsed as is without errors.

If you have samples that cause an error - please open the issue (but don't forget to add ddl example), I will be glad to fix it.

A lot of statements and output result you can find in tests on the github - https://github.com/xnuinside/simple-ddl-parser/tree/main/tests .

### How to install

```bash

    pip install simple-ddl-parser

```

## How to use

### Extract additional information from HQL (& other dialects)

In some dialects like HQL there is a lot of additional information about table like, fore example, is it external table, STORED AS, location & etc. This propertie will be always empty in 'classic' SQL DB like PostgreSQL or MySQL and this is the reason, why by default this information are 'hidden'.
Also some fields hidden in HQL, because they are simple not exists in HIVE, for example 'deferrable_initially'
To get this 'hql' specific details about table in output please use 'output_mode' argument in run() method.

example:

```python

    ddl = """
    CREATE TABLE IF NOT EXISTS default.salesorderdetail(
        SalesOrderID int,
        ProductID int,
        OrderQty int,
        LineTotal decimal
        )
    PARTITIONED BY (batch_id int, batch_id2 string, batch_32 some_type)
    LOCATION 's3://datalake/table_name/v1'
    ROW FORMAT DELIMITED
        FIELDS TERMINATED BY ','
        COLLECTION ITEMS TERMINATED BY '\002'
        MAP KEYS TERMINATED BY '\003'
    STORED AS TEXTFILE
    """

    result = DDLParser(ddl).run(output_mode="hql")
    print(result)
```

And you will get output with additional keys 'stored_as', 'location', 'external', etc.

```python

    # additional keys examples
  {
    ...,
    'location': "'s3://datalake/table_name/v1'",
    'map_keys_terminated_by': "'\\003'",
    'partitioned_by': [{'name': 'batch_id', 'size': None, 'type': 'int'},
                        {'name': 'batch_id2', 'size': None, 'type': 'string'},
                        {'name': 'batch_32', 'size': None, 'type': 'some_type'}],
    'primary_key': [],
    'row_format': 'DELIMITED',
    'schema': 'default',
    'stored_as': 'TEXTFILE',
    ... 
  }

```

If you run parser with command line add flag '-o=hql' or '--output-mode=hql' to get the same result.

Possible output_modes: ["mssql", "mysql", "oracle", "hql", "sql"]

### From python code

```python
    from simple_ddl_parser import DDLParser


    parse_results = DDLParser("""create table dev.data_sync_history(
        data_sync_id bigint not null,
        sync_count bigint not null,
        sync_mark timestamp  not  null,
        sync_start timestamp  not null,
        sync_end timestamp  not null,
        message varchar(2000) null,
        primary key (data_sync_id, sync_start)
    ); """).run()

    print(parse_results) 

```

### To parse from file

```python
    
    from simple_ddl_parser import parse_from_file

    result = parse_from_file('tests/sql/test_one_statement.sql')
    print(result)

```

### From command line

simple-ddl-parser is installed to environment as command **sdp**

```bash

    sdp path_to_ddl_file

    # for example:

    sdp tests/sql/test_two_tables.sql
    
```

You will see the output in **schemas** folder in file with name **test_two_tables_schema.json**

If you want to have also output in console - use **-v** flag for verbose.

```bash
    
    sdp tests/sql/test_two_tables.sql -v
    
```

If you don't want to dump schema in file and just print result to the console, use **--no-dump** flag:


```bash
    
    sdp tests/sql/test_two_tables.sql --no-dump
    
```

You can provide target path where you want to dump result with argument **-t**, **--targer**:


```bash
    
    sdp tests/sql/test_two_tables.sql -t dump_results/
    
```

### More details

`DDLParser(ddl).run()`
.run() method contains several arguments, that impact changing output result. As you can saw upper exists argument `output_mode` that allow you to set dialect and get more fields in output relative to chosen dialect, for example 'hql'. Possible output_modes: ["mssql", "mysql", "oracle", "hql", "sql"]

Also in .run() method exists argument `group_by_type` (by default: False). By default output of parser looks like a List with Dicts where each dict == one entitiy from ddl (table, sequence, type, etc). And to understand that is current entity you need to check Dict like: if 'table_name' in dict - this is a table, if 'type_name' - this is a type & etc.

To make work little bit easy you can set group_by_type=True and you will get output already sorted by types, like:

```python

    { 
        'tables': [all_pasrsed_tables], 
        'sequences': [all_pasrsed_sequences], 
        'types': [all_pasrsed_types], 
        'domains': [all_pasrsed_domains],
        ...
    }

```

For example:

```python

    ddl = """
    CREATE TYPE "schema--notification"."ContentType" AS
        ENUM ('TEXT','MARKDOWN','HTML');
        CREATE TABLE "schema--notification"."notification" (
            content_type "schema--notification"."ContentType"
        );
    CREATE SEQUENCE dev.incremental_ids
        INCREMENT 10
        START 0
        MINVALUE 0
        MAXVALUE 9223372036854775807
        CACHE 1;
    """

    result = DDLParser(ddl).run(group_by_type=True)

    # result will be:

    {'sequences': [{'cache': 1,
                    'increment': 10,
                    'maxvalue': 9223372036854775807,
                    'minvalue': 0,
                    'schema': 'dev',
                    'sequence_name': 'incremental_ids',
                    'start': 0}],
    'tables': [{'alter': {},
                'checks': [],
                'columns': [{'check': None,
                            'default': None,
                            'name': 'content_type',
                            'nullable': True,
                            'references': None,
                            'size': None,
                            'type': '"schema--notification"."ContentType"',
                            'unique': False}],
                'index': [],
                'partitioned_by': [],
                'primary_key': [],
                'schema': '"schema--notification"',
                'table_name': '"notification"'}],
    'types': [{'base_type': 'ENUM',
                'properties': {'values': ["'TEXT'", "'MARKDOWN'", "'HTML'"]},
                'schema': '"schema--notification"',
                'type_name': '"ContentType"'}]}

```

### ALTER statements

Right now added support only for ALTER statements with FOREIGEIN key

For example, if in your ddl after table defenitions (create table statements) you have ALTER table statements like this:

```sql

ALTER TABLE "material_attachments" ADD FOREIGN KEY ("material_id", "material_title") REFERENCES "materials" ("id", "title");

```

This statements will be parsed and information about them putted inside 'alter' key in table's dict.
For example, please check alter statement tests - **tests/test_alter_statements.py**


### More examples & tests

You can find in **tests/** folder.

### Dump result in json

To dump result in json use argument .run(dump=True)

You also can provide a path where you want to have a dumps with schema with argument .run(dump_path='folder_that_use_for_dumps/')

## Supported Statements

- CREATE TABLE [ IF NOT EXISTS ] + columns defenition, columns attributes: column name + type + type size(for example, varchar(255)), UNIQUE, PRIMARY KEY, DEFAULT, CHECK, NULL/NOT NULL, REFERENCES, ON DELETE, ON UPDATE,  NOT DEFERRABLE, DEFERRABLE INITIALLY

- STATEMENTS: PRIMARY KEY, CHECK, FOREIGN KEY in table defenitions (in create table();)

- ALTER TABLE STATEMENTS: ADD CHECK (with CONSTRAINT), ADD FOREIGN KEY (with CONSTRAINT), ADD UNIQUE, ADD DEFAULT FOR

- PARTITIONED BY statement

- CREATE SEQUENCE with words: INCREMENT, START, MINVALUE, MAXVALUE, CACHE

- CREATE TYPE statement:  AS ENUM, AS OBJECT, INTERNALLENGTH, INPUT, OUTPUT

- LIKE statement (in this and only in this case to output will be added 'like' keyword with information about table from that we did like - 'like': {'schema': None, 'table_name': 'Old_Users'}).

- TABLESPACE statement

- COMMENT ON statement

### HQL Dialect statements

- PARTITIONED BY statement
- ROW FORMAT
- STORED AS
- LOCATION, FIELDS TERMINATED BY, COLLECTION ITEMS TERMINATED BY, MAP KEYS TERMINATED BY

### MSSQL / MySQL/ Oracle

- type IDENTITY statement
- FOREIGN KEY REFERENCES statement
- 'max' specifier in column size
- CONSTRAINT ... UNIQUE, CONSTRAINT ... CHECK, CONSTRAINT ... FOREIGN KEY, CONSTRAINT ... PRIMARY KEY
- CREATE CLUSTERED INDEX

### Oracle

- ENCRYPT column property [+ NO SALT, SALT, USING]
- STORAGE column property

### TODO in next Releases (if you don't see feature that you need - open the issue)

1. Add support for GENERATED ALWAYS AS statement
2. Add support for CREATE TABLESPACE statement
3. Add support for properties for TABLESPACE like `TABLESPACE user_data ENABLE STORAGE IN ROW CHUNK 8K RETENTION CACHE`
4. Add support for statement CREATE DOMAIN
5. Add CREATE DATABASE statement support
6. Add more support for CREATE type IS TABLE (example: CREATE OR REPLACE TYPE budget_tbl_typ IS TABLE OF NUMBER(8,2);
7. Add support for MEMBER PROCEDURE, STATIC FUNCTION, CONSTRUCTOR FUNCTION,  in TYPE
8. Add support (ignore correctly) ALTER TABLE ... DROP CONSTRAINT ..., ALTER TABLE ... DROP INDEX ...
9. Add support for COMMENT ON statement

## non-feature todo

0. Provide API to get result as Python Object
1. Add online demo (UI) to parse ddl

### Historical context

This library is an extracted parser code from https://github.com/xnuinside/fakeme (Library for fake relation data generation, that I used in several work projects, but did not have time to make from it normal open source library)

For one of the work projects I needed to convert SQL ddl to Python ORM models in auto way and I tried to use https://github.com/andialbrecht/sqlparse but it works not well enough with ddl for my case (for example, if in ddl used lower case - nothing works, primary keys inside ddl are mapped as column name not reserved word and etc.).
So I remembered about Parser in Fakeme and just extracted it & improved. 

## Changelog
**v0.14.0**
1. Added support for CONSTRAINT ... PRIMARY KEY ...
2. Added support for ENCRYPT [+ NO SALT, SALT, USING] statements for Oracle dialect. All default values taken from this doc https://docs.oracle.com/en/database/oracle/oracle-database/21/asoag/encrypting-columns-tables2.html
Now if you use output_mode='oracle' in column will be showed new property 'encrypt'. 
If no ENCRYPT statement will be in table defenition - then value will be 'None', but if ENCRYPT exists when in encrypt property you will find this information:

{'encrypt' : {
    'salt': True,
    'encryption_algorithm': 'AES192',
    'integrity_algorithm': 'SHA-1'
    }}

3. Added support for oracle STORAGE statement, 'oracle' output_mode now has key 'storage' in table data defenition.
4. Added support for TABLESPACE statement after columns defenition

**v0.12.1**
1. () after DEFAULT now does not cause an issue
2. ' and " does not lost now in DEFAULT values

**v0.12.0**
1. Added support for MSSQL: types with 2 words like 'int IDENTITY', 
FOREIGN KEY REFERENCES statement, supported 'max' as type size, CONSTRAINT ... UNIQUE statement in table defenition,
CONSTRAINT ... CHECK, CONSTRAINT ... FOREIGN KEY
2. Added output_mode types: 'mysql', 'mssql' for SQL Server, 'oracle'. If chosed one of the above - 
added key 'constraints' in table defenition by default. 'constraints' contain dict with keys 'uniques', 'checks', 'references'
it this is a COSTRAINT .. CHECK 'checks' key will be still in data output, but it will be duplicated to 'constraints': {'checks': ...}
3. Added support for ALTER ADD ... UNIQUE
4. Added support for CREATE CLUSTERED INDEX, if output_mode = 'mssql' then index will have additional arg 'clustered'.
5. Added support for DESC & NULLS in CREATE INDEX statements. Detailed information places in key 'detailed_columns' in 'indexes', example: '
'index': [{'clustered': False,
                'columns': ['extra_funds'],
                'detailed_columns': [{'name': 'extra_funds',
                                        'nulls': 'LAST',
                                        'order': 'ASC'}],
6. Added support for statement ALTER TABLE ... ADD CONSTRAINT ... DEFAULT ... FOR ... ;

**v0.11.0**
1. Now table can has name 'table'
2. Added base support for statement CREATE TYPE:  AS ENUM, AS OBJECT, INTERNALLENGTH, INPUT, OUTPUT (not all properties & types supported yet.)
3. Added argument 'group_by_type' in 'run' method that will group output by type of parsed entities like: 
'tables': [all_pasrsed_tables], 'sequences': [all_pasrsed_sequences], 'types': [all_pasrsed_types], 'domains': [all_pasrsed_domains]
4. Type in column defenition also can be "schema"."YourCustomType"
5. " now are not dissapeared if you use them in DDL.

**v0.10.2**
1. Fix regex that find '--' in table names (to avoid issue with -- comment lines near string defaults)

**v0.10.1**
1. Added support for CREATE TABLE ... LIKE statement
2. Add support for DEFERRABLE INITIALLY, NOT DEFERRABLE statements

**v0.9.0**
1. Added support for REFERENCES without field name, like `product_no integer REFERENCES products ON DELETE RESTRICT`
2. Added support for REFERENCES ON statement

**v0.8.1**
1. Added support for HQL Structured types like ARRAY < STRUCT <street: STRING, city: STRING, country: STRING >>, 
MAP < STRING, STRUCT < year: INT, place: STRING, details: STRING >>, 
STRUCT < street_address: STRUCT <street_number: INT, street_name: STRING, street_type: STRING>, country: STRING, postal_code: STRING >

**v0.8.0**
1. To DDLParser's run method was added 'output_mode' argument that expect valur 'hql' or 'sql' (by default).
Mode change result output. For example, in hql exists statement EXTERNAL. If you want to see in table information 
is it EXTERNAL table or not - you need to set 'hql' output_mode.
2. Added suppport for hql EXTERNAL statement, STORED AS statement, LOCATION statement
3. Added suppport for PARTITIONED BY statement (for both hql & sql)
4. Added support for HQL ROW FORMAT statement, FIELDS TERMINATED BY statement, COLLECTION ITEMS TERMINATED BY statement, MAP KEYS TERMINATED BY statement

**v0.7.4**
1. Fix behaviour with -- in strings. Allow calid table name like 'table--name'

**v0.7.3**
1. Added support `/* ... */` block comments
2. Added support for Mysql '#' comments

**v0.7.1**
1. Ignore inline with '--' comments

**v0.7.0**
1. Redone logic of parse CREATE TABLE statements, now they parsed as one statement (not line by line as previous)
2. Fixed several minor bugs with edge cases in default values and checks
3. Added support for ALTER FOREIGN KEY statement for several fields in one statement

**v0.6.1**
1. Fix minor bug with schema in index statements

**v0.6.0**
1. Added support for SEQUENCE statemensts
2. Added support for ARRAYs in types
3. Added support for CREATE INDEX statements

**v0.5.0**
1. Added support for UNIQUE column attribute
2. Add command line arg to pass folder with ddls (parse multiple files)
3. Added support for CHECK Constratint
4. Added support for FOREIGN Constratint in ALTER TABLE

**v0.4.0**
1. Added support schema for table in REFERENCES statement in column defenition
2. Added base support fot Alter table statements (added 'alters' key in table)
3. Added command line arg to pass path to get the output results
4. Fixed incorrect null fields parsing

**v0.3.0**
1. Added support for REFERENCES statement in column defenition
2. Added command line