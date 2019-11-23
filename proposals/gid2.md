# Breaking the 64K limit

* **The next font format (OpenType 2?) should support > 64K glyphs in one font.**
* GID will be 32-bit.
* A “subset” table would be present to support legacy APIs that expect GID being 16-bit. It is a bijection between 16-bit “Legacy GID” and 32-bit “Real GID”.
* Legacy API implementations should reject “OT2” fonts if the subset table is absent – which means this font is designed exclusively for new API.
* A `STAT`-like mechanism will be used to “break down” an “OT2” font into a family of legacy-API-aware fonts.
* When processing in legacy software…
  * TTF component references – if still supported – will reference to Real GID rather than Legacy GID.
  * `CMAP`, `GSUB`, `COLR`, `SVG ` entries involving glyphs outside the down-level subset will be ignored.
  * `GPOS` will not be affected.
