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

Conceptually, this proposal changes how all mark attachment lookups behave. After
attachment, a relationship is kept between the mark and its base, be that a base
or a mark. If the base in such a relationship is subsequently
moved, then all the marks in attachment relationship with that base are
moved by the same amount. This is akin to moving other cursively attached glyphs 
when a cursively attached glyph is moved, except it is with regard to marks.
If a base in cursive relationship with other bases is
moved such that one of its attached marks is now positioned outside the cluster
region, the cluster does not need to be adjusted (as in bases or advances changed).
Glyphs that are in a positional relationship with a base (mark or base) but not
and attachment relationship, do not move when the base moves. Likewise, attached
glyphs do not move if the advance of the base is adjusted, but following non-attached
glyphs will move.

## Changes

> Define LookupFlags 0x0040 as the `SpacingAttach` flag

The `SpacingAttach` flag has meaning in the context of Cursive, MarkToBase,
MarkToLigature and MarkToMark attachment type lookups. In each of these cases
the attaching glyph is treated as though it were a spacing mark, even in a
Cursive attachment. For all other lookup types, the flag is ignored.

The effect of setting the flag on different attachment type lookups is based
on the x-position of the attachment point on the _mark_ glyph (`_P`) that is
the one that moves, its advance (`_A`). Also on the attachment point
x-position on the _base_ glyph (`P`), that is the one that does not move, and
its advance (`A`).

### Mark to Base (Type 4)

On attachment, the advance of the base is adjusted such that if `_A + P - _P > A`
then `A` becomes `_A + P - _P` and the advance on the mark (`_A`) is set
to 0. Likewise if the width of the diacritic to the left is greater than the
base, then the base is shifted. The shift is `_P - P`, if that value is
greater than 0. Shifting a base shifts all marks that are attached to it. An
alternative interpretation is that all marks following a base are also
shifted by the same amount. The base advance is also increased by the same
amount.

Shifting of a base glyph is achieved by increasing the advance on the most
immediately preceding base or ligature glyph. This is necessary in order that
the attachment points on the base do not become mispositioned.

If the base is cursively attached, then for the purposes of advance or shift
the advance is of the accumulated advance of all the glyphs cursively
attached. The measurement for `P` is increased by the advances of all the
glyphs up to the base glyph, in the cusrively attached cluster. When
processing right to left, appropriate care must be taken that the advance and
origin of the cluster are appropriately calculated with respect to the
attached mark.

### Mark to Ligature (Type 5)

This behaves in exactly the same way as for a Mark to Base attachment.

### Mark to Mark (Type 6)

Marks may attach to other marks. Here only the advance of the base mark is
adjusted by the attachment of a mark. As for the Mark to Base attachment, the
advance of the base mark may be adjusted if `_A + P - _P > A` in which case
it becomes `_A + P - _P`. No adjustment is made of the origin of the base
mark since there is no way of doing this without mispositioning attachment
points on the base mark.

### Cursive Attachment (Type 3)

Except for the impact on Mark to Base attachment above, cursive attachment
itself is unaffected by the SpacingAttach flag and it is ignored for this
type of lookup.

## Rationale

Some shapers zero their marks. This means the advance of the mark is set to
zero. This makes it hard to have a mark contribute to the space of a cluster.
For those shapers that do not zero their marks, calculating the impact of an
overlapping attachment on the advance of the mark is problematic, otherwise the
font has the job of zeroing its marks.

This added semantic can be enabled to help resolve the calculations needed to
account for protruding diacritics and ensuring appropriate spacing with minimal
complexity.
