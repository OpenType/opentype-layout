# Move Lookup

This is a proposal to add a GSUB lookup to support glyph movement.

The purpose of this lookup is to move a glyph relative to its current position
in the glyph string. The lookup also supports swapping two glyphs.

Most OpenType implmentations use a cluster model whereby glyphs that are
attached or are reordered in relation to each other are in the same cluster.
Therefore, if a glyph is moved across a cluster boundary, that cluster boundary
should be removed and the clusters merged.

## Changes

Add a new GSUB lookup with lookup type of 9. There is only one format for this
lookup type.

MoveLookupFormat1:

Type  | Name       | Description
----- | ---------- | -----------
uint8 | MoveFlags  | Flags governing the move
int8  | MoveOffset | Distance to move, may be negative

If the MoveOffset results in a position outside the glyph string or the absolute
values of MoveOffset is 0 or greater than 32, no action occurs and the lookup is
ignored.

MoveFlags bit enumeration:

Type | Name       | Description
---- | ---------- | -----------
0x01 | MoveThis   | Moves the current glyph by the given offset
0x02 | MoveOther  | Moves the glyph at the given offset to before the current glyph

Notice that if both bits are set, the moves are considered to happen in
parallel. If executed in series then the offset may need to be adjusted by 1 to
get the final positions correct.
