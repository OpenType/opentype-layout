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

If the MoveThis flag is set, then if MoveOffset is greater than 0, then the
current glyph is moved to be after the glyph at the given relative offset.
Likewise if MoveOffset is less than 0, then the current glyph is moved to be
before the glyph at the given relative offset.

If the MoveOther flag is set, then if MoveOffset is greater than 0, then the
other glyph is moved to be after the current glyph. If MoveOffset is less than
0, then the other glyph is moved to be before the current glyph.

Notice that if both bits are set, the moves are considered to happen in
parallel and the two glyphs are swapped.


## Rationale

The need for a move semantic can make things much easier when dealing with
diacritic stacking when the diacritics do not stack vertically. For example, in
the Myanmar script, the medial ra (`U+103C) is shaped to be before the base
character it surrounds. The lower dot (`U+1037`) is shaped after the base, but
needs in some contexts to attach to the medial ra. It is not possible to attach
two marks with an intervening base (despite flags implying that). Therefore it
is necessary to contextually reorder the lower dot to before the base to achieve
this.

One approach to nastaliq nukta attachment attaches the nuktas in a cluster to a
following bari yeh rather than to their bases. This involves moving the nuktas
to follow the bari yeh.

Moving glyphs is possible now, but is hard work. To move the `b` in `axb` to
`bax` one could do the following:

````
lookup 1:
    a'lookup 2 x b
    b a x'lookup 3 b

lookup 2:
    sub a by b a

lookup 3:
    sub x b by x

````

It is easy to introduce problems in this example, that require the insertion of
extra marking glyphs and their removal to achieve a simple glyph move.
