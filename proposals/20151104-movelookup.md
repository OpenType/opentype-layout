# Move Lookup

This is a proposal to add a GSUB lookup to support glyph movement.

The purpose of this lookup is to move a glyph relative to its current position
in the glyph string. The lookup also supports swapping two glyphs.

Most OpenType implmentations use a cluster model whereby glyphs that are
attached or are reordered in relation to each other are in the same cluster.
If a glyph is moved across a cluster boundary, that cluster boundary
should be removed and the clusters merged.


## Changes

Add a new GSUB lookup with lookup type of 9. There is only one format for this
lookup type.

MoveLookupFormat1:

Type   | Name        | Description
-----  | ----------  | -----------
uint16 | SubstFormat | Format identifier-format = 1
Offset | Coverage    | Offset to a coverage of possible glyphs to move
uint16 | MoveFlags   | Flags governing the move
int8   | MoveOffset  | Distance to move, may be negative

The `Coverage` table specifies whether the glyph is to processed or not. The
`MoveOffset` is a signed offset specifying how many glyphs separate the
*this* and *other* glyphs. Glyphs that are skipped due to processes such
as mark skipping are not included in the offset.

MoveFlags bit enumeration:

Type   | Name        | Description
------ | ----------  | -----------
0x0001 | CopyThis    | Copies the current glyph at the given offset
0x0002 | DelThis     | Deletes the current glyph at its current position
0x0004 | BeforeThis  | At the given offset means before the glyph at that position
0x0008 | WithThis    | Incorporate marks associated with this glyph in the action
0x0010 | CopyOther   | Copies the other glyph to the position of this glyph
0x0020 | DelOther    | Deletes the other glyph
0x0040 | BeforeOther | Specifies whether to copy before or after this glyph
0x0080 | WithOther   | Incorporate marks associated with the other glyph in the action
0x0100 | matchEOS    | End of string constitutes a match

Consider first the *this* glyph which is the one matched by the lookup. The
`CopyThis` flag specifies that the glyph will be copied to the given offset.
If `BeforeThis` is set then it will be copied to before the glyph at the specified
offset. If not set then it will be copied after the glyph. If `WithOther` is set
then after the glyph means after any following associated skipped marks to the
other glyph. The `WithThis` flag indicates whether the associated skipped marks with the
*this* glyph are also copied with the glyph.

The `DelThis` flag specifies that after the copy (if done) the glyph,
and according to `WithThis` any associated skipped marks, are to be deleted.

A matrix can be made from `CopyThis` and `DelThis`:

Copy | Del  | Description
---- | ---- | -----------
0    | 0    | No action
0    | 1    | Delete
1    | 0    | Copy
1    | 1    | Move

The *other* glyph is handled correspondingly using the `CopyOther`, `DelOther`,
`BeforeOther` and `WithOther` flags.

If `matchEOS` is set and on scanning (due to mark filtering) the end of the string
is encountered, then it is considered to be counted. If the offset is such that this
would then be a good place to move something then that move will happen. One cannot
insert after the final EOS or before the initial EOS, therefore if the `BeforeThis`
is set wrongly, not action occurs. No *other* type actions occur with an EOS. They
are ignored.

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
