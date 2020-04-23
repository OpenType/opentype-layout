# `LABL` — Human-readable glyph labels

_By Adam Twardoch on 23 April 2020_

This is a proposal that I circulated in 2013 on the OpenType list. It had almost no response, but time passes and times change, so I’d like to pitch it again.

This time, I’d like to suggest that the table could be adopted by the OpenType format officially, or could be unofficially adopted by some tool and client vendors.

I’d like to propose a new SFNT table `LABL`, which will map glyph IDs to `name` table records which contain human-readable labels for the glyphs.

Below is a short introduction of the idea, followed by a draft proposal of the actual table structure, along with some examples, and a loose commentary.

## Rationale

In the past years, we have observed a true surge in *icon fonts*, where primarily web designers have been putting graphical symbols into fonts, and using them as UI web elements. The idea, of course, isn’t new. Quite likely, it’s been pioneered by Microsoft in their Marlett font which was used to draw certain UI elements in Windows 95.

For many years, there’s been a large number of symbol or dingbat fonts on the market, here’s just a [small sample](http://myfonts.us/a3rlZP). But it’s really around 2013 where “symbol fonts” seem to have taken off properly, with [FontAwesome](https://fontawesome.com/), Google’s [Material Icons](https://material.io/resources/icons/), Apple’s [SF Symbols](https://developer.apple.com/design/human-interface-guidelines/sf-symbols/overview/), and various collections like [IcoMoon](https://icomoon.io/), [Iconify](https://iconify.design/), [Fontello](http://fontello.com/), [Fontastic](http://fontastic.me/) and many others.

One problem with symbol fonts is that OpenType hasn’t proposed a sensible method to include human-readable descriptions of the glyphs included in the font.

I’d like to propose a simple idea how to solve this problem.

## `LABL` table: Label to Glyph Index Mapping Table

This table defines the mapping of human-readable names (or “labels”) to the glyph index values used in the font.

The purpose of this table is to provide application developers with the ability to present to users meaningful human-readable labels for glyphs, especially if the glyphs are non-textual. Application developers could also utilize this table to produce an alternative input method where the user could type in a portion of a label and then, the input text would be searched for in the `LABL` table, and matching glyphs from the current font (or from a selection of fonts) could be presented to the user for final input. Also, the labels could be used to aid accessibility by providing a plain-text description of otherwise graphical glyphs.

The `LABL` table may contain more than one subtable, in order to support more than one glyph labeling scheme (or “vocabulary”).

The overall structure of the `LABL` table mimicks the structure of the [`cmap`](https://docs.microsoft.com/en-us/typography/opentype/spec/cmap) table closely, and it re-uses some of the formats defined by the `cmap` table.

While the `cmap` table maps numerical character codes (Unicode codepoints or other numerical codes) to glyph indices, the `LABL` table maps [`name`](https://docs.microsoft.com/en-us/typography/opentype/spec/name) table record indices to glyph indices.

The following `cmap` subtable formats are permissible in the `LABL` table:

- [Format 0](https://docs.microsoft.com/en-us/typography/opentype/spec/cmap#format-0-byte-encoding-table) (Byte encoding table), with a slightly modified behavior (see
  NOTE below)
- [Format 4](https://docs.microsoft.com/en-us/typography/opentype/spec/cmap#format-4-segment-mapping-to-delta-values) (Segment mapping to delta values) \*
- [Format 6](https://docs.microsoft.com/en-us/typography/opentype/spec/cmap#format-6-trimmed-table-mapping) (Trimmed table mapping)
- Potentially also other `cmap` subtable formats, if considered useful.

## `LABL` table structure

The `LABL` Table is organized as follows:

### `LABL` Header

| Type   | Name      | Description                           |
| ------ | --------- | ------------------------------------- |
| USHORT | version   | Table version number (0).             |
| USHORT | numTables | Number of mapping tables that follow. |

### `LABL` Mapping Record

The `LABL` table header is followed by an array of mapping records that specify the particular mapping and the offset to the subtable for that mapping. The number of mapping records is numTables. A mapping record entry looks like:

| Type   | Name       | Description                                                           |
| ------ | ---------- | --------------------------------------------------------------------- |
| USHORT | platformID | Platform ID, always set to 5.                                         |
| USHORT | encodingID | Platform-specific encoding ID ("Vocabulary ID").                      |
| ULONG  | offset     | Byte offset from beginning of table to the subtable for this mapping. |

### `LABL` Subtable Header

Each mapping subtable has the following format:

| Type   | Name     | Description                                  |
| ------ | -------- | -------------------------------------------- |
| USHORT | format   | Format number, set to 0, 4 or 6.             |
| USHORT | length   | This is the length in bytes of the subtable. |
| USHORT | language | always set to 0.                             |

Then, the mapping of numerical IDs (which refer to the `name` table IDs) to glyph IDs follows, using structures defined by the `cmap` subtable formats 0, 4 or 6.

_NOTE: For the `LABL` subtable format 0, the value 256 is added to each index of the mapping array to retrieve the corresponding `name` table ID. So item 0 of the array corresponds to name table ID 256, item 1 corresponds to name table ID 257 and so forth. For other subtable formats, the numerical codes correspond to name table IDs directly._

### Platform ID, Encoding ID, Language ID

The Platform ID and Encoding ID values defined in the `LABL` table must correspond directly to the Platform ID and Encoding ID values defined in the `name` table. The Language ID in the `LABL` subtables is always set to 0, but in the `name` table, language-specific IDs are used.

#### Platform ID

A new Platform ID = 5 is defined for the purpose of glyph labeling. The `LABL` table always uses the Platform ID 5. Glyph labels defined in the `name` table must also use the Platform ID 5 accordingly.

#### Language ID

The Language ID in each `LABL` subtable must always be set to 0.

The Language IDs for the `name` table records which are referred to by the `LABL` table must use the same mechanism as the _platform ID = 3_ language identifiers, i.e. either the Windows Language IDs or, if `name` table format 1 is used, also the language tags.

For language-neutral labels, the `name` table records should use Language ID 0×0409 (U.S. English).

#### Encoding ID

In the context of the `LABL` table, the Encoding ID represents a **“Vocabulary ID”**.

- Vocabulary ID 0 is defined as “text contents”.
- Vocabulary ID 1 is defined as “private vocabulary”.
- Vocabulary ID 2 is defined as “vendor-specific vocabulary”.
- Vocabulary IDs > 2 are registered by the specification maintainer.

A label defined for the Vocabulary ID 0 defines the actual text which the labeled glyph represents, expressed as a Unicode UTF-16BE string.

By default, Vocabulary ID 0 labels are language-neutral, so they use Language ID 0×0409, however, localized labels are permissible. _If glyphs are mapped in the Unicode `cmap` table (3.1 or 3.10), it’s usually not necessary to provide labels. However, text glyphs only accessible via OpenType Layout features or other such mechanisms may use labels. from the Vocabulary ID 0._

Labels defined for the Vocabulary ID 1 can be formed freely by any font vendor, and do not need to adhere to any rules, conventions or standards.

Labels defined for the Vocabulary ID 2 can be formed freely by any font vendor, but the assumption is made that each font vendor (as identified by the achVendID code in the OS/2 table) maintains some sort of vocabulary to which the defined labels adhere.

Labels defined for the Vocabulary IDs > 2 need to be formed according to the vocabulary maintained by the registered entity.

Some Vocabulary IDs which might be proposed for registration:

- Vocabulary ID 3: Wikipedia (http://www.wikipedia.org/)

Any label mapped to a glyph and registered in the Vocabulary ID 3 (Wikipedia) should be spelled exactly like the title of the Wikipedia article which corresponds to that label in the appropriate language.

- Vocabulary ID 4: The Noun Project (http://thenounproject.com/)

Any label mapped to a glyph and registered in the Vocabulary ID 4 (The Noun Project) should be spelled exactly like the title of the entry on The Noun Project which corresponds to that label in the appropriate language.

- Vocabulary ID 5: The Medieval Unicode Font Initiative (http://www.mufi.info/)

Labels would be formed according to the “descriptive name” used within the Medieval Unicode Font Initiative.

- Vocabulary ID 6: SIL (http://scripts.sil.org/SILPUAassignments)

Labels would be formed according to the “descriptive name” used within SIL, especially for glyphs not accessible through OpenType Layout but instead accessible through the SIL Graphite layout system.

### Name records for glyph labels

Each `name` table record for a glyph label is encoded using Unicode UTF-16BE.

Unless there are other recommendations for a particular Vocabulary ID, it is recommended that for U.S. English labels, no leading or trailing spaces, should be used, common words be spelled in all-lowercase (while proper nouns or abbreviations using the appropriate normal-text casing), and little or no punctuation be used unless necessary.

Within each vocabulary, multiple `name` records can point to the same glyph, thus allowing one glyph to have multiple labels. It is up to the application developer how to prioritize looking up within multiple vocabularies and how to present user-readable labels to the user if a glyph has
multiple labels assigned.

## Examples

### Example 1. “Basketball” glyph

Let’s assume that the glyph with the glyph ID 34 represents a ball for playing basketball. In that case:

The `LABL` subtable with the PID 5 and EID 1 (= private vocabulary) could contain an entry mapping glyph ID 34 to the `name` table record 290.

A “private vocabulary” record could exist in the `name` table (NID 290, PID 5, EID 1, LID 0×0409) with the contents “basketball”, conforming with the “all lowercase” recommendation labels cited above.

An additional `LABL` subtable with the PID 5 and EID 3 (= Wikipedia) could exist that would contain an entry mapping glyph ID 34 to the `name` table record 268.

Then, the `name` table record could exist (NID 268, PID 5, EID 3 = Wikipedia, LID 0×0409 = U.S. English), and with the following contents (without the quotes): “Basketball (ball)”. This is because that is the title of the English Wikipedia article describing the ball for playing basketball: http://en.wikipedia.org/wiki/Basketball_(ball)

Another `name` table record could exist (NID 268, PID 5, EID 3 = Wikipedia, LID 0×0407 = German) with the contents “Basketball (Sportgerät)” as this is the title of the corresponding German Wikipedia article: http://de.wikipedia.org/wiki/Basketball_(Sportger%C3%A4t)

Finally, an additional `LABL` subtable with the PID 5 and EID 4 (= The Noun Project) could exist that would contain an entry mapping glyph ID 34 to the `name` table record 332.

Another `name` table record could exist (NID 332, PID 5, EID 4 = The Noun Project, LID 0×0409), with the following contents: “Basketball”. This is because that is the title of the English entry on The Noun Project: http://thenounproject.com/noun/basketball/

### Example 2: “Soccer field” glyph

If the glyph with the glyph ID 35 represents a soccer field, then the `LABL` subtable with the PID 5 and EID 1 could contain an entry mapping glyph ID 34 to the `name` table record 291, the `LABL` subtable (PID 5, EID 3) could map the glyph to the `name` table record 453, and the `LABL` subtable (PID 5, EID 4) could map the glyph to the `name` table record 453 as well.

In that case: NID 291 PID 5 EID 1 LID 0×0409: “soccer field” NID 453 PID 5 EID 3 LID 0×0409: “Association football pitch” → http://en.wikipedia.org/wiki/Association_football_pitch NID 453 PID 5 EID 4 LID 0×0409: “Soccer Field” → http://thenounproject.com/noun/soccer-field/

Since the soccer field does not currently have a separate Wikipedia article, it would not be possible to construct a proper label in the Vocabulary ID 3, LID 0×0407.

### Example 3: “FontLab” logotype

If the glyph with the glyph ID 36 contains the logotype which represents the word “FontLab”, then that glyph will probably be accessible through the OpenType Layout feature “liga” or “dlig” as a ligature of the glyphs /F/o/n/t/L/a/b.

However, in addition, the `LABL` subtable with the PID 5 and EID 0 (= text contents) could exist, which could map the glyph ID 36 to the `name` ID 259. In that case, the `name` table could contain the record NID 259 PID 5 EID 0 LID 0×0409: “FontLab”.

If EID 0 labels (“text contents”) are included for all glyphs in a font which are not encoded in the `cmap` table directly, then applications could provide meaningful glyph input methods or labels for glyphs without having to reverse engineer GSUB substitutions. This technique could be used for glyph variants, ligatures or more complex “wordglyphs” or “logoglyphs”.

## Discussion

In the above proposal, some portions (such as the concept of Vocabulary IDs, and in particular the registered Vocabulary IDs and the “text contents” EID 0), are optional, and could be “done away with” if the community deems it too complex. I’m kind of keen on at least keeping two EIDs: 0 for “text contents” and 1 for “private labels”.

I’m grateful to Laurence Penney for mentioning The Noun Project to me, and for his ideas that helped me formulate the “vocabulary” concept in this proposal.

Using fonts for non-textual content is a 500-years-old tradition. Borders, dingbats, mapping symbols and other kids of *repetitive* graphical units have a long tradition of being an equal part of the typesetting process just like textual characters. Also in the digital age, “symbol fonts” have always existed.

A digital font can be viewed as a “database” or “collection” or symbols, which is organized in some way and has established logical and spatial relations between the symbols.

The point of a *font* is not necessarily that it’s about _text_. Primarily, it’s about “movable type”, i.e. having a coordinated, automated system to reproduce repetitive graphical units on a surface.

So actually, using fonts for graphics _is_ very digital, especially if the graphics are repetitive symbols. If it’s your own self-portrait, then it’s not useful to be put inside of a font. But if it’s a graphical unit that appears more than once, or is supposed to interact in any significant way with text or other symbols — then fonts _are_ just about the right path.

In computer programming, collections of data items are traditionally accessed using two indexing systems: by number (lists, arrays) or by name (hash tables, dictionaries). It’s widely agreed that when you index by number (or numerical code), there is a need of some external entity to “explain” the encoding system. For that, we have the Unicode Standard.

But the Unicode Standard falls short of providing a *complete* solution, because it requires that a symbol is registered in Unicode, and that’s a long process. And it’s actually quite OK, I don’t see why all kinds of symbols should find their way into the Unicode Standard.

I’d go even further, and say that the Unicode Standard actually outreached its own goal a bit. I think that the notion of encoding _seven_ (or whatever) different kinds of right-pointing arrows is silly. Why not just one arrow? Or nineteen? Why seven? “RIGHT SQUIGGLE ARROW”, “RIGHT WAVE ARROW”, “NOTCHED LOWER RIGHT-SHADOWED WHITE RIGHTWARDS ARROW”, “BACK-TILTED SHADOWED WHITE RIGHTWARDS ARROW”, “HEAVY BLACK-FEATHERED RIGHTWARDS ARROW”… Ehem?

But OK. We do have the Unicode Standard. It’s not perfect, but it’s fine for the most parts. But it’s “lookup by number” — a notion which is good for some applications but not really useful for others.

Of course OpenType fonts do have the concept of “glyph names”, i.e. the PostScript glyph names -- but their role has been long overloaded, especially since the Adobe-recommended practice has been established where the names should mimick Unicode codepoints using the “uniXXXX” convention. So,
PostScript glyph names are not in any way descriptive, really. But the fact that in original PostScript glyphs were keyed by name rather than number is telling — it tells us about human nature.

### Humans like words better than they like numbers.

Computers, of course, like numbers better than they like words. But typography, digital text -- is actually _primarily_ a tool for _communications between humans_ these days. It’s far less about computers than some software engineers might think.

Even if SVG is supported in all browsers, this does not remove the usefulness of having glyph labels. First of all, SVG glyphs can be included in SFNT fonts (a proposal and an implementation already exist), so it seems obvious that people still are interested in having SVG symbols _organized_ in some way.

I think what you’re missing is that a font is not _just_ a collection of symbols. It’s a collection of symbols that, through layout systems, establishes logical and spatial relationships between these symbols. Inside a font, you can define what should happen if certain symbols occur in a sequence, under what circumstances different variants are used, and what is the spacing behavior of these symbols in relation to each other. That’s something you won’t easily implement in a cross-platform way if you just have a “bag of loose SVG graphics”. Also, fonts have (and will even more in future) a mechanism for choosing size-specific variants of the same symbol, so you won’t have to rely on linear scaling, which often produces optically sub-par results.

We have at least three layout systems on the market (OpenType Layout, AAT, SIL Graphite), and none of them provides a fully adequate mechanism to provide explanatory metadata to all kinds of glyph variants the font may have.

Glyph labeling is very useful for the textual context as well. Imagine a simple situation: you have a font which has two variants of the asterisk (`*`): one with six arms and another with five arms. You can encode one as a stylistic alternate of the other, but there isn’t really an easy way to provide the user with the information that the one glyph is “asterisk with six arms” and the other is “asterisk with five arms”.

The same goes for other kinds of specially-formed glyph variants, where the designer might want to embed some useful information about what particular glyphs are useful for, what’s their stylistic treatment etc.

And we have the “lookup by name” issue. That’s where the Vocabulary idea (which was suggested by Laurence) comes in: any entity could establish a Vocabulary and register it. Then, they could publish their vocabulary and the corresponding expected symbols. For example, an organization of map
rendering services could agree on a vocabulary for symbols used on maps. Then, font vendors could develop various fonts for use on maps, and if they label their glyphs using the mapping Vocabulary, the notion of switching the style of a map would be simple — the glyphs in a font could be looked up
using the mapping Vocabulary labels, and then, when the font is switched, a different set of symbols could be used. This could would be much more sensible to implement than doing all kinds of “corporate use of the PUA” kinds of hackery (which still could be done of course).

And finally — how the heck are you currently supposed to find a glyph which shows, say, a banana on MyFonts? There are fonts on MyFonts which have a glyph that shows a banana, I’m sure. But there is no way for the user to discover them now.

Of course font developers could make accompanying documents which describe to the user what each symbol is. But there is no standardized way to make such documents, and such documents do not travel with the font. The idea behind the `LABL` proposal is to embed this metadata inside of the font, so it doesn’t get “lost” on its travels.

Another example is mathematical typesetting: regardless of whether a typesetting engine uses the Microsoft MATH table or some TeX typesetting technique or yet another way — I think it’d be useful if the mathematical players in the field (e.g. STIX) created a vocabulary for glyphs used in math
typesetting, and embedded them into their fonts. Other mathematical font vendors could follow that. This would aid switching fonts, providing better fallback scenarios or even just developing new math fonts (because the labels would be helpful for other font developers to understand the nature of a particular glyph).

There is a large number of legitimate usage scenarios for fonts where labels are better than numbers. Either to look up glyphs by label, or to display the label to the user.

My proposal aims to be lightweight. It re-uses data structures from the `cmap` table, and makes use of the long-established `name` table, so implementers don’t need to develop practically any new code (at least in the portion of parsing the font). I tried to make the proposal not too much
over the top, yet I tried to ensure that several useful scenarios are catered for.

### Earlier proposals, `Zapf` table

Apple maintains a spec for the SFNT [`Zapf`](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6Zapf.html) table that partially has similar goals. I have reviewed the `Zapf` table spec before writing my own proposal. I always liked the name of the `Zapf` table and its general intent — but at the same time, I have always found the actual table format hard to understand.

There are multiple different structures (GlyphInfo, KindName, GroupInfo, GroupInfoGroup, NamedGroup, FeatureInfo), and I have a hard time imagining a sensible user interface for it. Also, the `Zapf` FeatureInfo structure seems to be closely tied to AAT. So, `Zapf` tries to expose metadata “for everything”: glyphs, glyph groups (classes) and typographic features (tied to AAT). By doing this, I think it shoots over its own goal a bit.

The `Zapf` table is almost 20 years old, and has a number of concepts which certainly are out of date (hardcoded four types of names: Apple name, Adobe name, AFII name, Unicode name). The table has been defined in pre-webfont, and even to some extent pre-web times.

With the recent discussion of color font formats, I’ve heard one recurring word of praise for the Microsoft proposal (`COLR`/`CPAL`): that it was _simple and lightweight_, therefore easy to implement. I have noticed that this aspect of the Microsoft proposal was almost universally praised, and this was what has motivated me to attempt a similar path with the `LABL` proposal. I have a feeling that the `Zapf` table is a tad too ambitious, and a bit
“over-engineered”, and therefore never really found wide adoption.

So, when writing the `LABL` proposal, I tried to learn from the lack of adoption of `Zapf` and from the overall warm welcome of the `COLR/CPAL` proposal. I tried to make the structure clean and simple to use.

I tried to create a structure that is a bit modular — e.g. the concept of registered Vocabulary IDs is sort of optional. If the community finds that aspect too much of a complication, it can be removed (leaving only one or two hardcoded Encoding IDs) without invalidating the entire structure. Analogically, if the community decides to drop that idea now but revisits it in future, the notion of adding an extra Encoding ID is easy and won’t break previous implementations.

Plus, the Vocabulary concept allows for a clean separation: each vocabulary exists in a separate LABL subtable, therefore a font developer can easily set up, say, 30 labels within one particular vocabulary, and then 150 labels within another vocabulary, independently of each other. This is something I learned to like about `cmap` — that each `cmap` subtable can really be handled separately. These days, only the cmap 3.x is of relevance, but the fact that you can “safely” add or remove 0.x, 1.x or 4.x cmap subtables without interfering with the 3.x subtable, and that these subtables could address different subsets of the glyphset, always was appealing to me.

The fact that pretty much every font editing tool already has a `name` table editor, and that all platforms have the ability to parse the `name` table makes the `LABL` proposal a rather low-hanging fruit.

Since the `LABL` table itself is really trivial in structure in itself, and re-uses `cmap` structures, also means that adding support for it will be very easy. For example, I imagine that adding `LABL` support to something like fontTools/TTX is a matter of 15 minutes work.

The only administrative overhead resulting from my proposal will be the maintenance of the list of registered Vocabulary IDs. But I believe it’s unlikely we’ll ever get more than a dozen of those, so it’s still quite “cheap”.

Of course, the maintenance of the actual vocabularies or “policing” semantic conformance of the labels to a particular vocabulary is completely out of scope of the spec — just like it is out of scope of the spec to ensure that, in a particular font, the glyph with the Unicode U+0041 really depicts an uppercase “A”.

Regards,
Adam Twardoch
