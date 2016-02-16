# Complex Contextual Chaining Lookup

This is a proposal to add a GSUB lookup to support complex contextual chaining including string permutation.


## Description

The complex contextual chaining lookup is a class based contextual GSUB lookup. The lookup is designed to only have one subtable. In the case of multiple subtables, each subtable is its own pass: equivalent to a full lookup.

### ComplexChainLookup1

Type   | Name        | Description
------ |-----------  |--------------------------
uint16 | SubstFormat | Format identifier-format = 1
Offset | ClassDef    | class table for glyphids to be matched, relative to the start of the subtable
uint8  | maxBackup   | Maximum string backup for matching
uint8  | minBackup   | Minimum string backup
uint8  | maxLoop     | Maximum number of iterations before progress must have been made
uint8  | reserved    |
Offset | ChainNode[] | Array of maxBackup - minBackup + 1 ChainNodes, relative to the start of the subtable

The `ClassDef` is a single class that categorises glyphs in the input string. The `maxClass` speeds up class
matching in the ClassNode array. Any class value greater than this is considered to be 0.

The `maxBackup` and `minBackup` values describe how processing occurs. The input glyph string is backed up
by maxBackup glyphs (skipping marks if specified, etc.) and if that is not possible then by as many as possible.
If this number is less than minBackup then the lookup is said to have failed and processing stops. The amount
of backup is kept and is used to modify indices in the ChainNode. Subtracting the minBackup from this backup
value gives the index in the ChainNode array which specifies the starting ChainNode for processing. Before
backing up, the class index for the current glyph is tested for 0. If it is 0, then processing is skipped
for this glyph and the match position is advanced.

As the lookup progresses through the string, each action is able to adjust the starting point for the next
match. This adjustment includes not advancing or even advancing backwards. To ensure that the lookup
does not process forever or explode its output, it keeps track of the furthest point in the input string that
has been reached. A counter counts each time a match starts and is reset when the furthest point is reached.
If the count reaches `maxLoop`, because progress is not being made, then the lookup jumps its processing
to the furthest point and continues from there, as a best attempt to recover.

### ChainNode

Type   | Name                 | Description
------ |-----------           |--------------------------
Offset | ChainAction          | Action for a final state. May be NULL
uint16 | numTransitions       | Number of transitions
struct | ClassNode[]          | Array of numTransitions ClassNode

A ChainNode represents both action and comparison. During matching the ClassNode array is searched for
a node corresponding to the class index of the current glyph in the string. If matched, the search
position in the input string is advanced and processing continues with the corresponding ChainNode.
This continues until no match occurs. If a ChainNode has no ChainAction then the engine is back
tracked to the previously matching ChainNode and so on until a ChainNode with a ChainAction is
encountered, or the beginning of the match is encountered. In effect a non-zero ChainAction offset
marks a node as being a final state.

If no action occurs for a given match, the string is advanced one position and matching is done
again.

### ClassNode

When matching, the classIndex of the glyph being tested is searched for in the list of ClassNodes. If
there is no corresponding ClassNode for the classIndex, then the match has failed. The only case when
this is not the case is if there is a ClassNode with a classIndex of 0xFFFF.

Type   | Name       | Description
------ |----------- |--------------------------
uint16 | classIndex | class index value to match
Offset | chainNode  | Offset to ChainNode from start of the lookup subtable, for this match

A `classIndex` of 0xFFFF is special. It is used as a default transition. In ChainNodes in the backup string the default action for many class
indices is to transition to the ChainNode for the next shorter backup ChainNode. Rather than having
to store entries for all the unspecified class indices, using 0xFFFF allows for a fallback and alleviates
the need to store so many entries.

### ChainAction1

Type   | Name                 | Description
------ |-----------           |--------------------------
uint16 | ActionFormat         | Format identifier-format = 1
uint8  | permuteIndex         | Index of the start of the string replaced by permutation
uint8  | permuteReplace       | Number of input indices to replace during permutation
uint8  | advance              | How far back to move start of next match
uint8  | numSubstLookupRecord | Number of SubstLookupRecords
struct | SubstLookupRecord[]  | Array of numSubstLookupRecord SubstLookupRecords
uint16 | permuteLength        | Number of indices in the permute string
uint8  | permute[]            | Array of permuteLength indices

Indices correspond to positions in the matched string according to a matched node, back from the end
of the string thus far matched. Thus index 1 is after index 2 in the glyph string. If mark skipping
is enabled, for example, it may be possible for there to be many glyphs between index 2 and index 1.

If the `permuteLength` of a ChainNode being actioned is non-zero then the input glyph string, starting
with the backup length index is replaced by a permutation of the input string. The `permute` string
lists a sequence of indices which are used to replace the string up to the `permuteReplace` index. An index
in the `permute` string may actually reference more than one glyph if mark skipping is turned on, for example.
The same index may occur multiple times in the permute string. If the `permuteLength` is such that it consumes
the final index (index of 0) then the "current position" of 0 is moved to be the end of the permuted string
as output.

The permuted string is then used as input to the substlookuprecord processing. If an index of 0 is
replaced with a longer string, the "current position" is advanced to be after the generated string
that replaces the final index.

After all processing, further processing of the string occurs with the string index after that
referenced by `advance`.

## Discussion

Since the lookup has complete control over the processing of the pass, it is possible for it
to track the making of progress and to ensure that a font doesn't cause an infinite loop or explode
its output unreasonably. To that end, implementations are advised to put a limit on the growth
of a glyph string during such a lookup and to exit early should that limit be exceeded.

Two other principles have influenced the design: size and speed. Using only one class table allows
the input glyph string to be remapped into class space to simplify comparison. Although this adds complexity
in handling the sublookups that get executed, given the ability this lookup provides to reprocess
the output from a previous match in this lookup. A class is also used over multiple coverage tables, to save space.

The string permutation obviates the need for a move lookup.

By making the chainNode offset relative to the subtable, it is possible to form loops in the
node chain. This allows for full DFA processing. The question is whether loops need to be supported
within the matched part of a string such that the action rules adapt accordingly. As it stands
the matched portion and the final part of the string will need to be fixed length. One option is to
facilitate the dropping of a marker in the input and then allowing indices to work relative to that
as well as to the end of the matched string.
