# Spacing Attachment

This is a proposal to extend the OpenType standard to support spacing marks.

## Introduction

One of the struggles in OpenType font development is when attaching a mark to
a base, the mark protrudes from the base and requires extra space either before
or after the base for it to protrude into, without colliding with another glyph.
The difficult problem is knowing what the size of this space should be given
it involves the size of the diacritic and its relative position on the particular
base it is attached to. Therefore, the value changes for every mark and every
base that can combine in such a way.

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

![Example](spacing_mark.png)

When a mark is attached to a cursively attached based, using spacing attachment,
the mark will not cause the positional relationship between the cursively attached
base and the base to which it is attached (or bases attached to it) to change.
Instead the the collection of mutually cursively attached bases and their marks
are treated as a visual cluster and the position of the root of the cursively
attached tree is adjusted to ensure that no marks in a spacing attachment relationship
extend to the left outside the bounds of the cluster. Likewise the advance of
the last base glyph in the cluster is adjusted to ensure that no marks in a
spacing attachment relationship extend to the right beyond the bounds of the
cluster.

Since the advance of the mark has been incorporated into the base,
the advance of the spacing mark is zeroed as it is attached.

## Changes

> Define LookupFlags 0x0040 as the `SpacingAttach` flag

The `SpacingAttach` flag has meaning in the context of MarkToBase,
MarkToLigature and MarkToMark attachment type lookups. In each of these cases
the attaching glyph is treated as though it were a spacing mark.
For all other lookup types, the flag is ignored.

The effect of setting the flag on different attachment type lookups is based
on the x-position of the attachment point on the _mark_ glyph (`_P`) that is
the one that moves, its advance (`_A`). Also on the attachment point
x-position on the _base_ glyph (`P`), that is the one that does not move, and
its advance (`A`).

There are various processing models, including keeping a full tree of mark
attachment relationships. The model described here is designed for a
"position and forget" model where marks are positioned but no relationship is
maintained. For this we introduce the concept of a shift attribute on the
_base_ glyph (`S`) and on the _mark_ (`_S`).

### Mark to Base (Type 4)

On attachment, the advance of the base is adjusted such that if `_A + S + P - _S - _P > A`
then `A` becomes `_A + S + P - _S - _P` and the advance on the mark (`_A`) is set
to 0. Likewise if the width of the diacritic to the left is greater than the
base, then the base is shifted. The shift is `_S + _P - S - P`, if that value is
greater than 0. 

In order to shift a base glyph, that is not cursively attached, there needs
to be an extra attribute on the glyph that holds the shift. Simply increasing
the advance on a previous glyph does not allow a future Mark to Base
attachment to know that this base already has extra space inserted in front
of it. After
all attachment is done the shift attribute can be used to either offset all
the glyphs in the cluster (base plus all following marks) and the advance of
the base glyph, or by increasing the advance on the preceding base or
ligature.

If the base is cursively attached, then for the purposes of advance or shift
the advance is of the accumulated advance of all the glyphs cursively
attached. The measurement for `P` is increased by the advances of all the
glyphs up to the base glyph, in the cursively attached cluster. Likewise the
advance of the base is the accumulated advances of all the glyphs following
the base, in the cursive cluster. Any increase in advance is applied to the
last glyph of the cursive cluster. This may affect the order in which glyphs
are attached in order to get expected behaviour. When processing right to
left, appropriate care must be taken that the advance and
origin of the cluster are appropriately calculated with respect to the
attached mark.

### Mark to Ligature (Type 5)

This behaves in exactly the same way as for a Mark to Base attachment.

### Mark to Mark (Type 6)

Marks may attach to other marks. Here attachment is much like for the Mark to
Base. Marks may have shifts and advances just like bases. The only difference
is that after all attachment is completed, the calculated extra shift of a mark (`S`) is ignored.

The effect of this approach to a long chain of stacked diacritics is that
they will have to be attached twice. The first pass is done in reverse order
with the latest mark attaching to the earlier in order to propagate all the
width and shift onto the bottom mark. Then the marks are attached in
conventional order. Long chains of spacing attachments are very rare.

### Cursive Attachment (Type 3)

The normal behaviour of cursive attachment is to set the advance of the second glyph to be the difference of the advance of the second and first glyph. Setting the space attach bit changes this behaviour such that if the resulting advance of the second glyph is < 0, it is set to 0.

Notice where a base character is cursively attached to another base, for purposes of spacing attachment, the base is considered to be attached to the other base as if it were a mark. Thus extra space only occurs to the left of the first glyph in a cluster chain or after the last and not within the chain.

## Rationale

Some shapers zero their marks. This means the advance of the mark is set to
zero. This makes it hard to have a mark contribute to the space of a cluster.
For those shapers that do not zero their marks, calculating the impact of an
overlapping attachment on the advance of the mark is problematic, otherwise the
font has the job of zeroing its marks.

This added semantic can be enabled to help resolve the calculations needed to
account for protruding diacritics and ensuring appropriate spacing with minimal
complexity.
