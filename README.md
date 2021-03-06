# JSON Multi-Table data format - representing relational databases in plain text


| semantic version | updated    | authors          |
|------------------|------------|------------------|
| 1.1.0            | 2021-01-14 | Enrico Giampieri |


The goal is to introduce a detailed data format built on top of the excellent
[Jsonlines](https://jsonlines.org/) and [ndjson](https://github.com/ndjson/ndjson-spec).

this format is designed to replace traditional spreadsheets and
folders of csv with non trivial relationships among them.

The JSON format allows the inclusion of complex data and metadata, alongside
strict formatting and therefore easy parsing.

it places itself somewhere in between a csv, a spreadsheet and a database.

## Better than CSV
Advantages over CSV:
* single, well specified format
* can hold multiple tables in a single file
* objects for headers allow naturally to include metadata aside of columns names
* can include data such as numbers, strings, missing values and more complex data (any json can be a value)
* standardized encoding
* extremely simple implementation of a standardized reader

A common practice to include the bare minimum of information in a csv is to
include a first line with the column names.
Due to the limitations of the format, the homogeinety of the data, there is no
simple way to distinguish these first line from the data, and this is what limit
the possibility to use the csv for multi-table saving.

## Simpler than a database
Advantages over a database
* simple and plain data format (text), that allow long term safety
* being text can be easily kept under version control and diffed
* human readable and editable
* cross-platform, and easy to implement reader means that can be read and operated basically anywhere
* ease of implementation means no lock-in to any specific software

JMT can be easily imported and exported to plain database structure.
It could potentially hold all the information present in a database such as indexes, views and so one by leveraging a metadata table,
albeit this behavior is not standardized.

## Relationship with spreadsheets
A good analogy for JMT is to view it as a text-based spreadsheet data format.
Being text based guarantess that there will be no vendor lock-in, and the metadata availability

## Relationship with HDF5
JMT does not directly support random data access or binary compression, so it can't be used as a compressed and fast data access sudh as hdf5.
On the other end, being text based reduce the chance of catastrophic data corruption on edit and program locking to the single implementation of the parser.

To store binary data and be able to perform random access, it is not trivial, as
it requires some indexing.
Due to the text-based of the data, there is no fixed size of the data itself.
There are two possible solutions:
* using fixed width spacing for certain table's arrays
* use an indexing table at the end of the file and reading it backward

I would recommend the second option, as the first one can be very unstable
while processing the data.

## 1. Introduction

### 1.1 About
The proposal aim to define a way to exploit the ndjson/jsonlines format
to represents multiple tabular databasets in a single text file 
in a human readable format.


#### 1.2 Terminology
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", 
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to 
be interpreted as described in RFC 2119. \[[RFC2119]\]

## 2. Example JMT

    {"columns": ["name", "age"], "name": "people"}
    ["Albert", 21]
    ["Barbara", 45]
    {"columns": ["name", "pet specie", "pet name"], "name": "pets"}
    ["Albert", "cat", "meow"]
    ["Albert", "cat", "purr"]
    ["Barbara", "dog", "woof"]

## 3. Functional Specification

### 3.1 Serialization

Each JSON text MUST conform to the \[[RFC7159]\] standard and MUST be written 
to the stream followed by the newline character `\n` (0x0A).
The newline character MAY be preceded by a carriage return `\r` (0x0D). 
The JSON texts MUST NOT contain newlines or carriage returns.

All serialized data MUST use the UTF8 encoding.

* Each JSON MUST be either:
    - an object
    - an array
    - a string
* the strings-only JSON lines MUST be ignored and can be treated as comments
* The serialized data MUST start with one object
* Every object MUST be followed by one or more arrays
    - all these arrays MUST have the same number of elements
* Every object MUST contains one name representing the concept of *name of columns* with an array as value
    - the RECOMMENDED name is "columns"
    - the associated array MUST have the same lenght of the following arrays
    - the values of the associated array MUST be all strings
* Every object MUST contains one name representing the concept of *name of table* with an string as value
    - the RECOMMENDED name is "name", and is RECOMMENDED to be unique in the file
* it is RECOMMENDED that every object contains one name "types"
    - the value associated SHOULD be an object, with the column names as values
    - the values associated to each name are representative of the data type of the column
    - it is RECOMMENDED that the data types are expressed in string with the standard JSON definition of types
* it is RECOMMENDED to use whitespaces to indent the arrays, to provide visual clarity to the structures
* it is RECOMMENDED to use an empty line to visually separate the tables to provide visual clarity

### 3.2 Parsing

The parser MUST accept newline as line delimiter `\n` (0x0A) as well as 
carriage return and newline `\r\n` (0x0D0A). 

Whitelines in the line, before or after the JSON structure, should be ignored.

If the JSON text is not parsable, the parser SHOULD raise an error.
The parser MUST ignore empty lines, e.g. `\n\n`.

### 3.3 MediaType and File Extensions

The MediaType \[[RFC4288]\] for Newline Delimited JSON 
SHOULD be _application/x-jmt_.

When saved to a file, the file extension SHOULD be _.ndjson_.

## 4. Example of Parsing

A simple parser in python can be implemented in few lines of code:

```python
import itertools as it
import operator as op
from functools import partial
def get_tables(stream):
    # utility functions for type testing later on
    instance_of = lambda types, obj: isinstance(obj, types)
    is_obj_or_arr = partial(instance_of, (dict, list))
    is_arr = partial(instance_of, list)
    # remove the empty lines
    lines = filter(None, map(str.strip, stream))
    # transform them in json
    structs = (json.loads(line) for line in lines)
    # keep only objects and arrays
    good_elements = filter(is_obj_or_arr, structs)
    # remove the array that are at the beginning
    good_structs = it.dropwhile(is_arr, good_elements)
    # group together all the objects and all the arrays
    grouped = it.groupby(good_structs, type)
    # drop the data type, keep only the data, we know they are alternating
    grouped_simple = map(list, map(op.itemgetter(1), grouped))
    # pair them, this will also remove any trailing objects with no following arrays
    paired = zip(*([grouped_simple] * 2))
    # if there are multiple objects (technically a mistake) keep only the last one
    paired_single_obj = ({'info': k[-1], 'data':d} for k, d in paired)
    # generate the final data structure from the data
    final = {table['info']['name']: table for table in paired_single_obj}
    return final
```

that can be used as such

```python
data = StringIO("""
    [1, 2]
    {"columns": ["a", "b"], "name": "foo"}
    [1, {"a": 2}]
    [3, {"a": 4}]
    [5, {"a": 6}]
    {"extraneous object": "remove!"}
    {"columns": ["c", "d"], "name": "bar"}
    [2, [1, 0]]
    [4, [3, 2]]
    [7, [6, 5]]
    {"a": "b"}
    """)
from pprint import pprint
pprint(get_tables(data))
```

that returns:
```python
{'bar': {'data': [[2, [1, 0]], [4, [3, 2]], [7, [6, 5]]],
         'info': {'columns': ['c', 'd'], 'name': 'bar'}},
 'foo': {'data': [[1, {'a': 2}], [3, {'a': 4}], [5, {'a': 6}]],
         'info': {'columns': ['a', 'b'], 'name': 'foo'}}}
```

## 5. Copyright

This specification is copyrighted by the authors named in section 4.1. 
It is free to use for any purposes commercial or non-commercial.

### 5.1 Authors

The following authors are responsible for the JSON Multi-Table core-specification:

* Enrico Giampieri

## A. References

### A.1 Normative

[RFC2119]: http://www.ietf.org/rfc/rfc2119.txt "RFC 2119 - Key words for use in RFCs to Indicate Requirement Levels"
\[RFC2119\]: RFC 2119 - Key words for use in RFCs to Indicate Requirement Levels

[RFC7159]: http://www.ietf.org/rfc/rfc7159.txt "RFC 7159 -  The JavaScript Object Notation (JSON) Data Interchange Format"
\[RFC7159\]: RFC 7159 -  The JavaScript Object Notation (JSON) Data Interchange Format

[RFC4288]: http://www.ietf.org/rfc/rfc4288.txt "RFC 4288 - Media Type Specifications and Registration Procedures"
\[RFC4288\]: RFC 4288 - Media Type Specifications and Registration Procedures
