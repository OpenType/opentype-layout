# OpenType Documentation Needs

This document lists known documentation needs regarding the behaviour of OpenType based shaping.
Some behaviors are different for different shapers, and these differences should also be noted.

## Marks

### Zeroing Marks
When are marks zeroed? Before GPOS or after GPOS runs?

### Adjusting Base
After a mark is attached to a base glyph, what is the effect on the position of the mark of
changing the offset or advance of the base glyph?

## Ligatures

Ligatures are involved with ligature attachment. They are formed as part of a GSUB ligature
lookup.

### Are Ligature Components Bases?
For a ligature component that is not a mark, will a mark attach to the component as a base, using
Mark Attachment?

### How are Ligature Components formed and unformed?
In a 1 to many lookup how are marks attached?

#### How does a glyph become a ligature component?
What kind of lookup and which glyphs become ligature components

#### How does a ligature component lose its component status?
Once a glyph is a ligature component, is it possible to turn it back into a normal base/mark?

## Cursive Attachment

### Cursive attachment of marks

What happens? What happens if the first glyph is subsequently adjusted.

## Cluster Definitions

### What is the cluster definition for each shaping engine including DFLT

