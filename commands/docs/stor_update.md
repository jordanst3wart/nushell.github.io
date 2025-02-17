---
title: stor update
categories: |
  database
version: 0.90.0
database: |
  Update information in a specified table in the in-memory sqlite database.
usage: |
  Update information in a specified table in the in-memory sqlite database.
feature: default
---
<!-- This file is automatically generated. Please edit the command in https://github.com/nushell/nushell instead. -->

# <code>{{ $frontmatter.title }}</code> for database

<div class='command-title'>{{ $frontmatter.database }}</div>

## Signature

```> stor update {flags} ```

## Flags

 -  `--table-name, -t {string}`: name of the table you want to insert into
 -  `--update-record, -u {record}`: a record of column names and column values to update in the specified table
 -  `--where-clause, -w {string}`: a sql string to use as a where clause without the WHERE keyword


## Input/output types:

| input   | output |
| ------- | ------ |
| nothing | table  |

## Examples

Update the in-memory sqlite database
```nu
> stor update --table-name nudb --update-record {str1: nushell datetime1: 2020-04-17}

```

Update the in-memory sqlite database with a where clause
```nu
> stor update --table-name nudb --update-record {str1: nushell datetime1: 2020-04-17} --where-clause "bool1 = 1"

```
