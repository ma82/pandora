# Pandora Specs

## High-level description

Pandora takes a json-like tree made of primitive type leafs, lists and dicts and generates

- a collection of tables indexed by `path`
  - columns of primitive types with values taken from tree leaves
  - `nodeHash` field corresponding to the merkle tree node hash
    - useful for spotting duplicates, at any level
    - not to be used as `id`
  - reproducibile `parentId`, `id` and `index` fields for each row

## Design choices

1. No-dependency core package
2. No schema management in the core: the target tables can be inconsistently typed and we let the user choose what to do with the inconsistencies by not resolving them
3. `tables` function parameterised by arbitrary Leaf type, allowing for type-safe wrappers
4. The merge of the output of different records is left outside from the core (could be even left to the persistence layer)
5. We want to make sure that records from nodes with identical underlying data but from trees that differ elsewhere are duplicated, because
   - they might be part of different partitions in the target storage
   - we want to know the unique parent id for every node in O(1), without performing joins
   - we might want to delete some data (e.g. without affecting unrelated nodes)
   - we might want to consider these records as different when doing aggregations (e.g. count)
6. Collapsing tables at different paths is a responsibility left to post-processing
   - the uses chooses the Path => String function, allowing recursive data models to be managed

## Algorithm

We consider a single input json-like value at once and perform

- an initial pass where we compute the merkle-tree hashes bottom-up, and we annotate the tree with those
- another pass where we annotate each node and leaf of the tree with the `path`
- another top-down pass where we annotate every node with three values:
  - `parentId`
    - root node: `null`
    - other nodes: `parent.id`
  - `index`
    - root node: 0
    - based on the last component of the `path` for all the other nodes
  - `id = hash(parentId ++ index ++ nodeHash)`
    - `nodeHash` is needed to build the `id` for the root node, but we consider it at all levels just for consistency
- a traversal where we extract only the annotations, and we accumulate the actual tables
  - when the node is an array
    - we add a table for the node path, if missing
    - we build a single record with index `null` and no values
  - when the node is an object
    - we add a table for the node path, if missing
    - we build a single record with index `null`
  - when the node is a leaf
    - if its parent is an array
      - we add a record with non-`null` index to a new table
    - if its parent is an object
      - we add our value to the parentId record in the field we compute from the path

## Future work

1. Allow putting leaves in purely content-addressed `hash`-indexed tables instead of `id`-indexed tables
  - reference those values using hashes in the `id`-indexed leaf record
  - could be useful when you have large "flat" nodes, i.e. with lots of primitive type fields
  - usually not what you want, as it requires expensive joins for simple aggregations
  - requires garbage collection algorithm to handle
2. Reduce the number of non-necessary tables

## Examples

    root_table_name = "root"
    separator = "_"
    single_value_field_name = "value"
    array_elements_table_name_suffix = "elems"

### 1

    {"x"-> [4,2,4], y -> "ciao" }

    =>

    table "root"

    id   | parent_id | node_hash | index  | y
    bdbd | <null>    | zzbz      | <null> | "ciao"
    
    table "root_x"
    
    id   | parent_id | node_hash | index
    ccbd | bdbd      | iuim      | <null>

    table "root_x_elems"

    id   | parent_id | node_hash | index | value
    cppc | ccbd      | hbdh      | 0     | 4
    uhhy | ccbd      | zsds      | 1     | 2
    zaza | ccbd      | hbdh      | 2     | 4

### 2

    {"x" -> {"y" -> [], "z" -> {}}}
    {"x" -> {"a" -> [1,2,3]}, "w" -> 10}

    =>

    table "root"

    id   | w
    dqlw |
    ghgh | 10

    table "root_x"

    id   | parent_id | node_hash | index
    bfbf | dqlw      | zbzb      | <null>
    dsda | ghgh      | lkmx      | <null>

    table "root_x_y"

    id   | parent_id | node_hash | index
    gggz | bfbf      | jdsd      | <null>

    table "root_z"

    id   | parent_id | node_hash | index
    asds | dqlw      | poji      | <null>

    table "root_x_a"

    id   | parent_id | node_hash | index
    pcpc | dsda      | bcbc      | <null>

    table "root_x_a_elems"

    id   | parent_id | node_hash | index | value
    aasd | pcpc      | dsda      | 0     | 1
    qweq | pcpc      | dsda      | 1     | 2
    aasd | pcpc      | dsda      | 2     | 3

### 3

    {"x" -> {"y" -> [], "z" -> {}}}
    {"x" -> 10}

    =>

    table "root"

    id   | parent_id | node_hash | index  | x
    sdas | <null>    | hdus      | <null> | <null>
    cycy | <null>    | isup      | <null> | 10

    table "root_x"

    id   | parent_id | node_hash | index
    dsda | sdas      | ascd      | <null>

    table "root_x_y"

    id   | parent_id | node_hash | index
    jdsa | dsda      | jdjs      | <null>

    table "root_x_y_elems"

    id   | parent_id | node_hash | index | value

    table "root_x_z"

    id   | parent_id | node_hash | index
    dsae | sdas      | kiuy      | <null>

### 4

    [{"a" -> 1, "b" -> "x"}, {"a" -> 3, "c" -> false}]

    =>

    table "root"

    id   | parent_id | node_hash | index
    qwep | <null>    | nckj      | <null>

    table "root_elems"

    id   | parent_id | node_hash | index | a | b      | c
    pisd | qwep      | ncnx      | 0     | 1 | "x"    | <null>
    oiuw | qwep      | cxop      | 1     | 3 | <null> | false

### 5

    "x"

    =>

    table "root"

    id   | parent_id | node_hash | index

### 6

    [[1,2,3], [4,5,6]]

    =>

    table "root"

    id   | parent_id | node_hash | index
    xyxy | <null>    | asda      | <null>

    table "root_elems"

    id   | parent_id | node_hash | index
    baba | xyxy      | dasd      | 0
    hdhd | xyxy      | aspo      | 1

    table "root_elems_elems"

    id   | parent_id | node_hash | index | value
    dsda | baba      | oius      | 0     | 1
    sdff | baba      | iuoi      | 1     | 2
    poip | baba      | qweq      | 2     | 3
    jfij | hdhd      | puji      | 0     | 4
    zxza | hdhd      | oiuy      | 1     | 5
    huio | hdhd      | asda      | 2     | 6
