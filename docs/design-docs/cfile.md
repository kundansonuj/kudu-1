<!---
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
CFile is a simple columnar format which stores multiple related B-Trees.


File format
-----------------
```
<header>
<blocks>
<btree root blocks>
<footer>
EOF


Header
------

<magic>: see below
<header length>: 32-bit unsigned integer length delimiter
<header>: CFileHeaderPB protobuf


Footer
------

<footer>: CFileFooterPB protobuf
<magic>: see below
<footer length> (length of protobuf)


Magic strings
-------------
CFile v1: the string 'kuducfil'
CFile v2: the string 'kuducfl2'
```

The two versions are identical except where specifically noted in the source
code or this document.

==============================

Data blocks:

Data blocks are stored with various types of encodings.

* Prefix Encoding

Currently used for STRING blocks. This is based on the encoding used
by LevelDB for its data blocks, more or less.

Starts with a header of four uint32s, group-varint coded:
```
  <num elements>       \
  <ordinal position>   |
  <restart interval>   |  group varint 32
  <unused>             /
```
Followed by prefix-compressed values. Each value is stored relative
to the value preceding it using the following format:
```
  shared_bytes: varint32
  unshared_bytes: varint32
  delta: char[unshared_bytes]
```
Periodically there will be a "restart point" which is necessary for
faster binary searching. At a "restart point", shared_bytes is
0 but otherwise the encoding is the same.

At the end of the block is a trailer with the offsets of the
restart points:
```
  restart_points[num_restarts]:  uint32
  num_restarts: uint32
```
The restart points are offsets relative to the start of the block,
including the header.


* Group Varint Frame-Of-Reference Encoding

Used for uint32 blocks.

Starts with a header:
```
<num elements>     \
<min element>      |
<ordinal position> | group varint 32
<unused>           /
```
The ordinal position is the ordinal position of the first item in the
file. For example, the first data block in the file has ordinal position
0. If that block had 400 data entries, then the second data block would
have ordinal position 400.

Followed by the actual data, each set of 4 integers using group-varint.
The last group is padded with 0s.
Each integer is relative to the min element in the header.

==============================

Nullable Columns

If a column is marked as nullable in the schema, a bitmap is used to keep track
of the null and not null rows.

The bitmap is added the begininning of the data block, and it uses RLE.
```
  <num elements in the block>   : vint
  <null bitmap size>            : vint
  <null bitmap>                 : RLE encoding
  <encoded non-null values>     : encoded data
```
Data Block Example - 4 items, the first and last are nulls.
```
  4        Num Elements in the block
  1        Null Bitmap Size
  0110     Null Bitmap
  v2       Value of row 2
  v3       Value of row 3
```
==============================

Index blocks:

The index blocks are organized in a B-Tree. As data blocks are written,
they are appended to the end of a leaf index block. When a leaf index
block reaches the configured block size, it is added to another index
block higher up the tree, and a new leaf is started. If the intermediate
index block fills, it will start a new intermediate block and spill into
an even higher-layer internal block.

For example:
```
                      [Int 0]
           ------------------------------
           |                            |
        [Int 1]                       [Int 2]
    -----------------            --------------
    |       |       |            |             |
[Leaf 0]     ...   [Leaf N]   [Leaf N+1]    [Leaf N+2]
```

In this case, we wrote N leaf blocks, which filled up the node labeled
Int 1. At this point, the writer would create Int 0 with one entry pointing
to Int 1. Further leaf blocks (N+1 and N+2) would be written to a new
internal node (Int 2). When the file is completed, Int 2 will spill,
adding its entry into Int 0 as well.

Note that this strategy doesn't result in a fully balanced b-tree, but instead
results in a 100% "fill factor" on all nodes in each level except for the last
one written.

There are two types of indexes:

- Positional indexes: map ordinal position -> data block offset

These are used to satisfy queries like: "seek to the Nth entry in this file"

- Value-based indexes: reponsible for mapping value -> data block offset

These are only present in files which contain data stored in sorted order
(e.g key columns). They can satisfy seeks by value.


An index block is encoded similarly for both types of indexes:
```
<key> <block offset> <block size>
<key> <block offset> <block size>
...
   key: vint64 for positional, otherwise varint-length-prefixed string
   offset: vint64
   block size: vint32

<offset to first key>   (fixed32)
<offset to second key>  (fixed32)
...
   These offsets are relative to the start of the block.

<trailer>
   A IndexBlockTrailerPB protobuf
<trailer length>
```
The trailer protobuf includes a field which designates whether the block
is a leaf node or internal node of the B-Tree, allowing a reader to know
whether the pointer is to another index block or to a data block.
