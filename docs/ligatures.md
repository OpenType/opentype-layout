# Ligatures

This technical note is about how ligatures are formed and dealt with in OpenType. It
is particularly concerned with the two ligature and multiple substitution lookups
in GSUB.

It might seem obvious that when a ligature is formed, the components that were
constituent in forming the ligature would be considered ligature components. But what
is less obvious is that when a ligature is broken (using a multiple substitution),
the generated glyphs are considered components of the source ligature.

The difficulty arises with mark to base attachment. If a base is considered a ligature
component, then it cannot take part in mark to base attachment as a base. There are
two provisions for when this limit is relaxed: for a ligature with component index of 0,
or for a ligature base.

A ligature base is formed as the result of a ligature substitution, but only if there
is more than one source glyph to the ligature. You can't fool it by doing a 1:1 ligature.
Ligature components are numbered with indices according to where they occur in the ligature.
The simple approach is to say, the first glyph in a 1:n sequence has an index of 0 and
so on. But ligature sequences may combine and so a sequence may not start at 0.

## Reordering Example

As an example of how this works out in practise, consider two base glyphs, x & y. We
want to reorder xy to yx. Further let's insert some extra glyphs in between: w & z. Now
we want to reorder xwzy to yxwz. We will do this with a single contextual chaining rule
and a whole bunch of multiple and ligature substitutions.

A simple naive approach might be to do the following transformations:

> xwzy / x -> yx and zy -> z

This gives the desired result. But notice that in the multiple substitution, x is now the
second component and so has a component index of 1. This means that no diacritics can
attach to x. We have to do something more complicated:

> xwzy / x -> yxy; xy -> x; zy -> z

Here now the xy -> x step has made x into a ligature base and so now diacritics
can attach to it. In addition, all the glyphs in the sequence are either component 0 (for y) or a ligature
base (for x & z).

To achieve this, we would need the following fea:

```
lookup yxy {
  sub x by y x y;
} yxy;

lookup zy {
  sub x y by x;
  sub z y by z;
} zy;

lookup reorder {
  sub x' lookup yxy w' lookup zy z y' lookup zy;
} reorder;
```

Why is the zy lookup, used to map xy -> x; associated with the w glyph? The reason is that
the string grows by 2 glyphs as part of the first lookup and that we need the lookup
executed on the second glyph of the new string.

This leads to a further problem. If w & z are not present then our reordering reduces to:

> xy / x -> yx and xy -> x

For this we would need another lookup: xy that sub x by y x; rather than by y x y;

```
lookup yx {
  sub x by y x ;
} yx;

lookup yxy {
  sub x by y x y;
} yxy;

lookup zy {
  sub x y by x;
  sub z y by z;
} zy;

lookup reorder {
  sub x' lookup yxy w' lookup zy z y' lookup zy;
  sub x' lookup yx y' lookup zy;
} reorder;
```

And so the final result for a simple! reordering is 3 large lookups with entries
for all x and z, multiplied by the contents of y (3 lookups per y!) and then a chaining contextual
to hold it all together. A move lookup is so much simpler!

## DirectWrite

Testing on DirectWrite has turned up that breaking a ligature composed of a sequence as done in this
example doesn't work. I'm assuming the reasoning goes something like: it's all the same ligature so the
components remain. This is not helpful and creating a ligature out of a set of components should result
in components being renumbered.
