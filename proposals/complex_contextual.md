# Complex Contextual Chaining Lookup

This is a proposal to add a GSUB lookup to support complex contextual chaining including string permutation.

## Description

The complex contextual chaining lookup is a class based contextual GSUB lookup.

### ComplexChainLookup1

Type   | Name        | Description
------ |-----------  |--------------------------
uint16 | SubstFormat | Format identifier-format = 1
Offset | ClassDef    | class table for glyphids to be matched, relative to the start of the subtable
uint16 | maxClass    | Maximum class value in the class table
uint8  | maxLength   | Maximum string match length
uint8  | maxBackup   | Maximum string backup for matching
uint8  | minBackup   | Minimum string backup
uint8  | reserved    |
Offset | ChainNode[] | Array of maxBackup - minBackup + 1 ChainNodes, relative to the start of the subtable

The `ClassDef` is a single class that categorises glyphs in the input string. The `maxClass` speeds up class
matching in the ClassNode array. Any class value greater than this is considered to be 0.

The `maxLength` specifies the maximum sequence of nodes that can be matched in a single string. If a chainNode
sequence occurs that is maxLength then no further searching is done and the action or backtracking may occur.

The `maxBackup` and `minBackup` values describe how processing occurs. The input glyph string is backed up
by maxBackup glyphs (skipping marks if specified, etc.) and if that is not possible then by as many as possible.
If this number is less than minBackup then the lookup is said to have failed and processing stops. The amount
of backup is kept and is used to modify indices in the ChainNode. Subtracting the minBackup from this backup
value gives the index in the ChainNode array which specifies the starting ChainNode for processing.

### ChainNode1

A ChainNode represents both action and comparison. During matching the ClassNode array is searched for
a node corresponding to the class index of the current glyph in the string. If matched, the search
position in the input string is advanced and processing continues with the corresponding ChainNode.
This continues until either `maxLength` nodes have been processed or no match occurs. The ChainNode
at this point is then actioned. If `permuteLength` and `numSubstLookupRecord` are both 0 then the
input string is backtracked to the last match position and matching ChainNode and this is actioned,
and so on until a ChainNode has something to do.

If the `permuteLength` of a ChainNode being actioned is non-zero then the input glyph string, starting
with the backup length index is replaced by a permutation of the input string. The `permute` string
lists a sequence of indices which are used to replace the string up to the `permuteReplace` index. Notice
that the `permuteReplace` and `permute` indices are made absolute by adding the backup length to them. An index
in the `permute` string may actually reference more than one glyph if mark skipping is turned on, for example.
The same index may occur multiple times in the permute string.

The permuted string is then used as input to the substlookuprecord processing. The indices in the
SubstLookupRecords as increased by the backup length reference the glyph string after permutation.

After all processing, further processing of the string occurs with the string index after that
referenced by `advance`. The value of `advance` must be at least that of `permuteReplace` and of
any index in a SubstLookupRecord. The final index is the sum of `advance` and the backup length.

Type   | Name                 | Description
------ |-----------           |--------------------------
uint8  | advance              | index of final element in input string
uint8  | permuteLength        | Number of indices in the permute string
uint8  | permuteReplace       | Number of input indices to replace during permutation
uint8  | numSubstLookupRecord | Number of SubstLookupRecords
uint16 | numTransitions       | Number of transitions
struct | ClassNode[]          | Array of numTransitions ClassNode
struct | SubstLookupRecord[]  | Array of numSubstLookupRecord SubstLookupRecords
uint8  | permute[]            | Array of permuteLength indices


ClassNode :

Type   | Name       | Description
------ |----------- |--------------------------
uint16 | classIndex | class index value to match
Offset | chainNode  | Offset to ChainNode from start of the ChainNode, for this match

A `classIndex` of 0 is special. Since there is no difference between lack of a glyph entry in a class
table and a class index of 0, all such glyphs are considered to not be matched by the engine. If such
a glyph occurs it causes a match failure. The use, therefore, of a 0 `classIndex` in a ClassNode can
be used as a default transition. In ChainNodes in the backup string the default action for many class
indices is to transition to the ChainNode for the next shorter backup ChainNode. Rather than having
to store entries for all the unspecified class indices, using 0 allows for a fallback and alleviates
the need to store so many entries.

## Discussion

Much of the design of the complex contextual chaining lookup is concerned with ensuring that a font
cannot be designed that can lock up or blow up (make an infinitely long string). To that end,
`advance` is constrained such that the processing cannot move backwards or anything be inserted
after its position.

Two other principles have influenced the design: size and speed. Using only one class table allows
the input glyph string (of calculable length from `maxLength`) to be remapped into class space to
simplify comparison. A class is also used over multiple coverage tables, to save space.

The string permutation obviates the need for a move lookup.

An alternative approach to adding the back length to all indices, is to track when engine is
processing backup chainNodes. Once the transition is made out of backup then that is considered the
index base.

