# Polygon Kerning

## Introduction

Kerning is primarily a visual activity. The current mechanisms for describing kerning are all string based and for complex strings, kerning can become prohibitively hard to describe. An alternative approach is to adjust the spacing between clusters based on some notional spacing bubble that can be described around the glyphs. This lookup uses such outlines to adjust the spacing between clusters by adjusting the space between bases.

The 'bubble' is described using a simple linear polygon (hence the lookup name). If a glyph has no polygon described for it, then its bounding box is used.

## Changes

This new lookup has one format. The outline is described using a sequence of anchors with an implicit closure of the outline from the last anchor to the first. Anchors are used to allow for font variation and, less likely, device adjustment.

A base plus all attached diacritics and cursively attached bases with their diacritics and cursively attached bases and so on, constitues a cluster. Two clusters are compared and the space between them adjusted such that the space is minimised but that no two bubble polygons from the two clusters, overlap. This space is increased in the presence of space glyphs (glyphs with no outline) by the advance widths of all the intervening space glyphs between the two clusters.

If the xmargin value is specified, then the minimum space between two clusters must be greater or equal to the margin. Thus intervening spaces may make the xmargin redundant. The ymargin effectively increases the top and lowers the bottom of one of the bubble polygons during comparison between two glyphs by the ymargin amount.

The maxOverlap value, if other than 0xFFFF (which indicates it is to be ignored) limits the negative difference between the right hand side of the first cluster and the left hand side of the second cluster. This limits how much two clusters may overlap.

PolyKernLookupFormat1:

Type   | Name       | Description
-------|------------|------------
uint16 | posFormat  | Format identifier: format = 1.
uint16 | glyphCount | glyphid of highest glyph with a polygon specified.
uint16 | flags      | bit 0: 1=16 bit offsets, else 32 bit offsets.
uint16 | maxOverlap | Max distance a cluster may kern into another. 0xFFFF for no limit.
uint16 | xmargin    | Minimum horizontal space between clusters.
uint16 | ymargin    | Minimum vertical space between clusters.
Offset16 or Offset 32 | offsets[glyphCount+1] | Offsets to start of point list for the given glyph.
Offset16 or Offset 32  | anchors[]   | Anchor point for a corner of the bubble outline.

It is possible for interaction to be across more than two clusters. For example, if a cluster completely encapsulates a folloowing cluster, then the cluster after that interacts with both previous clusters. One limit on cluster overlap is that the left hand of one cluster may not be to the left of the left hand side of a previous (in left to right terms) cluster. This stops unattached diacritics floating off.

## Issues

There is no base coverage table in this lookup. The problem with using a coverage table is that while it may, possibly speed up the hunt for cluster bases, the problems caused by missed cluster bases is greater than the value of speeding up the process. Consider the string 'abc' what is the semantics of a base coverage table of 'ac'? We nee the 'b' to kern and if it is ignored, does that mean we would see 'c' overlay 'b'? Should such a coverage table include spaces or not?
