---
title: try
categories: |
  core
version: 0.90.0
core: |
  Try to run a block, if it fails optionally run a catch block.
usage: |
  Try to run a block, if it fails optionally run a catch block.
feature: default
---
<!-- This file is automatically generated. Please edit the command in https://github.com/nushell/nushell instead. -->

# <code>{{ $frontmatter.title }}</code> for core

<div class='command-title'>{{ $frontmatter.core }}</div>

## Signature

```> try {flags} (try_block) (catch_block)```

## Parameters

 -  `try_block`: Block to run.
 -  `catch_block`: Block to run if try block fails.


## Input/output types:

| input | output |
| ----- | ------ |
| any   | any    |

## Examples

Try to run a missing command
```nu
> try { asdfasdf }

```

Try to run a missing command
```nu
> try { asdfasdf } catch { 'missing' }
missing
```
