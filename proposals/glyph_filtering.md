# Glyph Filtering

The current mechanism for mark filtering provides a useful level of increased
flexibility in processing lookups. This proposal extends that to include non-marks.

## Change

> Define LookupFlag 0x0020 to be `UseFiltering`

If set `UseFiltering` extends the mark filtering concept to include non marks. The
flag works in conjunction with the `UseMarkFilteringSet` flag to specify the
semantics of `MarkFilteringSet`.

Filtering | MarkFiltering | Description
--------- | ------------- | -----------
0         | 0             | No action
0         | 1             | Specifies marks not to skip, while skipping marks
1		  | 0             | Specifies glyphs to skip, while skipping any glyphs
1         | 1             | Specifies glyphs not to skip, while skipping any glyphs

If in addition to `UseFiltering` any of the `IgnoreBaseGlyphs`, `IgnoreLigatures` or
`IgnoreMarks` are set, then all of the corresponding classes of glyphs will be
skipped in addition to those specified by `UseFiltering`.
