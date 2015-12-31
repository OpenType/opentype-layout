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

Type   | Name        | Description
-----  | ----------  | -----------
uint16 | SubstFormat | Format identifier-format = 1
Offset | ClassDef    | Offset to glyph ClassDef table-from beginning of Substitution table. May be NULL
uint8  | MoveFlags   | Flags governing the move
int8   | MoveOffset  | Distance to move, may be negative

If the MoveOffset results in a position outside the glyph string or the absolute
values of MoveOffset is 0 or greater than 32, no action occurs and the lookup is
ignored.

MoveFlags bit enumeration:

Type | Name       | Description
---- | ---------- | -----------
0x01 | MoveThis   | Moves the current glyph by the given offset
0x02 | MoveOther  | Moves the glyph at the given offset to before the current glyph
0x04 | MoveLimit  | Only move if the glyph at MoveOffset is in class 2
0x08 | MoveScan   | Scans up to MoveOffset, skipping any glyphs in class 3

If the MoveThis flag is set, then if MoveOffset is greater than 0, then the
current glyph is moved to be after the glyph at the given relative offset.
Likewise if MoveOffset is less than 0, then the current glyph is moved to be
before the glyph at the given relative offset.

If the MoveOther flag is set, then if MoveOffset is greater than 0, then the
other glyph is moved to be after the current glyph. If MoveOffset is less than
0, then the other glyph is moved to be before the current glyph.

Notice that if both bits are set, the moves are considered to happen in
parallel and the two glyphs are swapped.

The MoveLimit and MoveScan flags are used in conjunction with the ClassDef table,
which if not present, are ignored. The classes in the ClassDef table have a fixed
meaning:

* Class 1: Only glyphs in class 1 will be moved

* Class 2: Glyphs will only swap or reorder before/after glyphs of class 2, if the
  MoveLimit flag is set.

* Class 3: If the MoveScan flag is set, rather than simply checking and reordering at the given
  MoveOffset, the lookup will scan from the given glyph in class 1 up to the
  MoveOffset in the direction of MoveOffset. The scan will skip any glyphs of
  class 3 until a glyph of class 2 is encountered, in which case the glyph
  will be reordered to before or after that glyph. If the scan reaches
  MoveOffset without encountering a glyph from class 2, then no action occurs.
  If MoveLimit is not set, then all glyphs not in class 3 are considered to be
  in class 2.

If the ClassDef offset is NULL then any glyph will move and the MoveLimit and
MoveScan flags are ignored.

## Rationale

There are a number of cases where a reordering semantic is needed where the
reordering is not emulating what a higher level shaping engine should be doing:

* The need for a move semantic can make things much easier when dealing with
diacritic stacking when the diacritics do not stack vertically. For example, in
the Myanmar script, the medial ra `U+103C` is shaped to be before the base
character it surrounds. The lower dot `U+1037` is shaped after the base, but
needs in some contexts to attach to the medial ra. It is not possible to attach
two marks with an intervening base (despite flags implying that). Therefore it
is necessary to contextually reorder the lower dot to before the base to achieve
this.

* One approach to nastaliq nukta attachment attaches the nuktas in a cluster to a
following bari yeh rather than to their bases. This involves moving the nuktas
to follow the bari yeh.

* In Thai, the sara am character `U+0E23` is decomposed, for rendering into `U+0E4D`,
which is a diacritic, and `U+0E32` for the spacing component. This character may be
preceeded by a tone mark `U+0E48`..`U+0E4B` but the tone mark is to be rendered above
the `U+0E4D`. Since attachment only occurs backwards, the tone mark needs to be
reordered after the `U+0E4D`.

* Also in Thai, one minority language uses a combining macron below as a consonant modifier.
Due to the relative canonical combining orders, this character will end up following
a lower vowel (sara u, sara uu) when it needs to be rendered closer to the consonant
than the vowel. The two glyphs need to be reordered. 

Moving glyphs is possible now, but is hard work and slow. To move the `b` in `axb` to
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

For complex ranges of x, where x is a string and other substitutions may occur
within the string, these lookups can become complex and interact in complex ways, sometimes
needing special marker glyphs to be inserted and deleted. This also slows down
shaping due to the added contextual lookups.

If this proposal is implemented, the above lookups become:

````
lookup 1:
    swap a' x b'
````

The inclusion of the more complex class table and scanning semantics allow the lookup
to be used outside of a direct reference from a chaining contextual substution.
