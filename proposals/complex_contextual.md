# Complex Contextual Chaining Lookup

This is a proposal to add a GSUB lookup to support complex contextual chaining including string permutation.


## Description

The complex contextual chaining lookup is a class based contextual GSUB lookup. The lookup is designed to only have one subtable. In the case of multiple subtables, each subtable is its own pass: equivalent to a full lookup.

### ComplexChainLookup1

Type     | Name         | Description
-------- |------------- |--------------------------
uint16   | SubstFormat  | Format identifier-format = 1
Offset   | ClassDef     | class table for glyphids to be matched, relative to the start of the subtable
uint8    | maxBackup    | Maximum string backup for matching
uint8    | minBackup    | Minimum string backup
uint8    | maxLoop      | Maximum number of iterations before progress must have been made
uint8    | reserved     |
uint16   | backupNode[] | Array of maxBackup - minBackup + 1 ChainNode references
uint16   | numNodes	    | Number of ChainNodes
Offset32 | ChainNode[]  | Array of offsets to ChainNodes

The `ClassDef` is a single class that categorises glyphs in the input string.

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
uint8  | actionid			  | Action identifier for a final node
uint16 | numActions			  | Number of Action offsets.
Offset | ChainAction[]        | Action for a final state.
uint16 | numTransitions       | Number of transitions
struct | ClassNode[]          | Array of numTransitions ClassNode

A ChainNode represents both action and comparison. During matching the ClassNode array is searched for
a node corresponding to the class index of the current glyph in the string. If matched, the search
position in the input string is advanced and processing continues with the corresponding ChainNode.
As it goes the engine collects actions to process at each node. This continues until no match occurs.
The engine backtracks until it finds a final node, one with a non-zero `actionid`.

Once a final state is selected for execution, all the collected actions with the same `actionid` as
the final state, are executed at their corresponding positions in the order they were collected.

The `ClassNode` is an array of ClassNodes sorted by `classIndex`.

### ClassNode

When matching, the classIndex of the glyph being tested is searched for in the list of ClassNodes. If
there is no corresponding ClassNode for the classIndex, then the match has failed. The only case when
this is not the case is if there is a ClassNode with a classIndex of 0xFFFF.

Type   | Name       | Description
------ |----------- |--------------------------
uint16 | classIndex | class index value to match
uint16 | chainNode  | chainNode index to use on match

A `classIndex` of 0xFFFF is special. It is used as a default transition. Rather than having
to store entries for all the unspecified class indices, using 0xFFFF allows for a fallback and alleviates
the need to store so many entries.

### ChainAction

Type   | Name     | Description
------ |--------- |--------------------------
uint8  | chainid  | Action chain number of this action
uint8  | distance | How far back to process
uint16 | lookup   | Lookup id to execute

The `distance` correspond to a position in the matched string according to a matched node, back from the end
of the string thus far matched. Thus distance 1 is after distance 2 in the glyph string. If mark skipping
is enabled, for example, it may be possible for there to be many glyphs between distance 2 and distance 1.

The lookup specifies the lookup id to execute at the specified string position. The following are
special lookup ids that are reserved for other actions:

Id     | Description
------ | -----------
0xFFFF | Start new match here after action processing

For a lookup id of 0xFFFF, the last executed action will give the final result.

> Need to describe what happens when a lookup changes the length of the processed string.

After a final state action is completed the engine restarts following that point in the string.

## Discussion

Since the lookup has complete control over the processing of the pass, it is possible for it
to track the making of progress and to ensure that a font doesn't cause an infinite loop or explode
its output unreasonably. To that end, implementations are advised to put a limit on the growth
of a glyph string during such a lookup and to exit early should that limit be exceeded.

Two other principles have influenced the design: size and speed. Using only one class table allows
the input glyph string to be remapped into class space to simplify comparison. Although this adds complexity
in handling the sublookups that get executed, given the ability this lookup provides to reprocess
the output from a previous match in this lookup. A class is also used over multiple coverage tables, to save space.

By making the chainNode offset relative to the subtable, it is possible to form loops in the
node chain. This allows for full DFA processing. The action_chain mechanism is primarily of use
for looped ChainNodes to allow processing within a klein star sequence.

With care it is possible for a compiler to add transitions between ChainNodes that will incorporate
the default behaviour of advancing, without having to have the engine restart a match, and
so have to back up for the backtrack. But this is not a requirement.

