# Spacing Marks

This is a proposal to extend the OpenType standard to support spacing marks.

Spacing marks are marks considered to have extent. When attached to a base or
another mark, such marks cause the extent of the base to be adjusted to ensure
that the combined cluster includes the extent of the mark in its attached
position. For example, if a mark is attached such that it overhangs to the right
of the base, the advance
of the base is extended to include the extent of the mark, and the mark itself
is given a zero advance. Likewise if such a mark were attached such that the
origin of the positioned mark were to the left of the origin of the base,
the origin of the cluster would be shifted back to include the origin of
the mark, while the offset from the origin of the base would be equally adjusted
to keep it in its same relative position.

In addition, since the advance of the mark has been incorporated into the base,
the advance of the spacing mark is zeroed as it is attached.

## Changes

### LookupTable

The lookuptable is extended to support multiple flags in anticipation of extra
flag needs, and a single flag is added which has meaning for Attachment lookups :

LookupTable:

Type   | Name          | Description
------ | ------------- | -----------
uint16 | LookupType    | Different enumerations for GSUB and GPOS
uint16 | LookupFlag    | Lookup qualifiers
uint16 | SubTableCount | Number of subtables in this lookup
Offset | Subtable[SubTableCount] | Array of offset to Subtables from beginning of Lookup table
uint16 | MarkFilteringSet | Only present if UserMarkFilteringSet of LookupFlag is set
uint16 | ExtraFlag     | More lookup qualifiers, only present if ExtraFlags of LookupFlag is set

LookupFlag bit enumeration:

Type   | Name                 | Description
------ | -------------------- | -----------
0x0001 | RightToLeft          | Only used for cursive attachment
0x0002 | IgnoreBaseGlyphs     | If set, skips over base glyphs
0x0004 | IgnoreLigatures      | If set, skips over ligatures
0x0008 | IgnoreMarks          | If set, skips over all combining marks
0x0010 | UserMarkFilteringSet | If set, use mark filtering
0x0060 | Reserved             | Set to zero
0x0080 | ExtraFlags           | If set, the ExtraFlag field is included and used
0xFF00 | MarkAttachmentType   | If not zero, skips over all marks of attachmant type different from specified.

ExtraFlag bit enumeration:

Type   | Name                 | Description
------ | -------------------- | -----------
0x0001 | SpacingMarks         | Apply spacing mark semantics when attaching glyphs in attachment lookups
0x7FFE | Reserved             | Set to zero
0x8000 | Reserved             | Set to zero, reserved for future extra flag extensions

The `SpacingMarks` flag has meaning in the context of Cursive, MarkToBase,
MarkToLigature and MarkToMark attachment type lookups. In each of these cases
the attaching glyph is treated as though it were a spacing mark, even in a
Cursive attachment. For all other lookup types, the flag is ignored.

## Rationale

Some shapers zero their marks. This means the advance of the mark is set to
zero. This makes it hard to have a mark contribute to the space of a cluster.
For those shapers that do not zero their marks, calculating the impact of an
overlapping attachment on the advance of the mark is problematic, otherwise the
font has the job of zeroing its marks.

This added semantic can be enabled to help resolve the calculations needed to
account for protruding diacritics and ensuring appropriate spacing with minimal
complexity.

## Comments

This approach to adding a single bit to a lookup is complicated by the desire to
leave space for future bits in the main LookupFlag.

The Reserved bits in the LookupFlag enumeration will probably be used for
directionality filtering. The need here is to ensure that the flags are
extensible into the future.

Where fields are unchanged from the original text, the original text description
should be used.
