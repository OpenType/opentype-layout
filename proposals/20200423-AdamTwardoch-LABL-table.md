# `LABL` — Human-readable glyph labels

_By Adam Twardoch on 23 April 2020_

This is a heavily-revised version of a proposal that I circulated in 2013 on the OpenType list. It had almost no response, but time passes and times change, so I’d like to pitch it again. This time, I’d like to suggest that the table could be adopted by the **OpenType format officially**, or could be **unofficially adopted by some tool and client vendors**.

I’d like to propose a new SFNT table `LABL`, which serves a function similar to the `name` table, but its records contain human-readable labels for the glyphs included in the font. Below is a short introduction of the idea, followed by a draft proposal of the actual table structure, along with some examples, and a loose commentary.

To discuss this, I suggest the [issue on my repo](https://github.com/twardoch/opentype-layout/issues/1).

## Rationale

> PostScript names are not enough, there are various legitimate cases where a font vendor would like to include human-readable descriptions for glyphs. My `LABL` proposal is very lightweight. It follows the structure of the `name` table so that all existing code that deals with the `name` table can be re-used with little or no adaptation. All all font parsers and tools can be made aware of the `LABL` table with little or no effort.

In the past years, we have observed a true surge in *icon fonts*, where primarily web designers have been putting graphical symbols into fonts, and using them as UI web elements. The idea, of course, isn’t new. Quite likely, it’s been pioneered by Microsoft in their Marlett font which was used to draw certain UI elements in Windows 95.

For many years, there’s been a large number of symbol or dingbat fonts on the market, here’s just a [small sample](http://myfonts.us/a3rlZP). But it’s really around 2013 where “symbol fonts” seem to have taken off properly, with [FontAwesome](https://fontawesome.com/), Google’s [Material Icons](https://material.io/resources/icons/), Apple’s [SF Symbols](https://developer.apple.com/design/human-interface-guidelines/sf-symbols/overview/), and various collections like [IcoMoon](https://icomoon.io/), [Iconify](https://iconify.design/), [Fontello](http://fontello.com/), [Fontastic](http://fontastic.me/) and many others.

One problem with symbol fonts is that OpenType hasn’t proposed a sensible method to include human-readable descriptions of the glyphs included in the font.

Also: most font editing apps (FontLab, Glyphs, RoboFont, FontForge) allow type designers to use glyph names during font development that don’t conform with the strict [Adobe Glyph Naming](https://github.com/adobe-type-tools/agl-aglfn/) recommendations. The development glyph names are stored inside development font formats such as [UFO](http://unifiedfontobject.org/), [`.glyphs`](https://github.com/schriftgestalt/GlyphsSDK/blob/master/GlyphsFileFormat.md) or [`.vfj`](https://github.com/kateliev/vfjLib/).

Some font vendors are interested to export those “development” glyph names into fonts. This proposal provides a simple place for development glyph names to exist within OpenType font files.

Some fonts are made for text but include many glyph variants or ligatures. It’s extremely cumbersome for app vendors to “map back” particular glyphs to their textual content. Client apps could use this table to map unencoded glyphs to their text representation in “Glyphs palette” types of scenarios.

## Proposal

I’d like to propose a simple idea how to solve this problem. The proposal for the `LABL` table (“Glyph labels table”) comes in two **variants**. Both variants can supply labels only for a few glyphs in the font, or for many, or for all of them:

- **Variant A** is extremely simple to implement: it’s identical in structure to the `name` table. However, it’s not so space-efficient, because each record includes the `platformID` and `encodingID` fields.
- **Variant B** is inspired by the existing `name` and `cmap` and `post` tables, but is not identical to them. It is more space-efficient.

**We should choose either Variant A or Variant B.**

# `LABL`: Glyph labels table

This table maps human-readable names (“labels”) to the glyph index values used in the font. The table may contain more than one glyph labeling scheme (“vocabulary”).

The purpose of this table is to provide application developers with the ability to present to users meaningful human-readable labels for glyphs, especially if the glyphs are non-textual. Application developers could also utilize this table to produce an alternative input method where the user could type in a portion of a label and then, the input text would be searched for in the `LABL` table, and matching glyphs from the current font (or from a selection of fonts) could be presented to the user for final input. Also, the labels could be used to aid accessibility by providing a plain-text description of otherwise graphical glyphs.

## `LABL` table Variant A

The structure of the labels table is identical to the OpenType naming table ([`name`](https://docs.microsoft.com/en-us/typography/opentype/spec/name). The labels table interprets some fields differently to the naming table.

### Labels table header

The labels table header is identical to the Naming table header. There are two formats for the Labels table, except that `LabelRecord` in used instead of a `NameRecord`:

- [Format 0](https://docs.microsoft.com/en-us/typography/opentype/spec/name#naming-table-format-0) uses platform-specific, numeric language identifiers.
- [Format 1](https://docs.microsoft.com/en-us/typography/opentype/spec/name#naming-table-format-1) allows for use of language-tag strings to indicate the language of strings.

Both formats include variable-size string-data storage, and an array of label records.

### LabelRecord: Label records

The label records follow the structure of the [Name Records](https://docs.microsoft.com/en-us/typography/opentype/spec/name#name-records) of the Naming table. However, the labels table uses slightly different identifiers.

| Type       | Name           | Description                                                      |
| ---------- | -------------- | ---------------------------------------------------------------- |
| _uint16_   | `vocabularyID` | Vocabulary ID instead of `name.platformID`                       |
| _uint16_   | `encodingID`   | Vocabulary-specific sub-identifier, instead of `name.encodingID` |
| _uint16_   | `languageID`   | Language ID, same as `name.languageID`                           |
| _uint16_   | `glyphID`      | Glyph ID, instead of `name.nameID`                               |
| _uint16_   | `length`       | String length (in bytes).                                        |
| _Offset16_ | `offset`       | String offset from start of storage area (in bytes).             |

By default, all strings are assumed to use Unicode **UTF-16BE** encoding.

- The `vocabularyID` identifier is used in the way discussed below.
- The `encodingID` may be used as a vocabulary-specific sub-identifier, for example for a major version of a particular vocabulary. If not meaningful, it should be `0`.

## `LABL` table Variant B

The `LABL` table borrows some concepts from the ([`name`](https://docs.microsoft.com/en-us/typography/opentype/spec/name) and [`cmap`](https://docs.microsoft.com/en-us/typography/opentype/spec/cmap) tables.

### Labels table header

The labels table is organized as follows:

| Type            | Name                          | Description                                                             |
| --------------- | ----------------------------- | ----------------------------------------------------------------------- |
| _uint16_        | `version`                     | Table version (`0`).                                                    |
| _uint16_        | `numTables`                   | Number of subtables.                                                    |
| _Offset32_      | `stringOffset`                | Offset to start of storage area (from start of table).                  |
| _LabelSubtable_ | `labelSubtable[count]`        | The label subtables where count is the number of subtables.             |
| _uint16_        | `langTagCount`                | Number of language-tag records.                                         |
| _LangTagRecord_ | `langTagRecord[langTagCount]` | The language-tag records where `langTagCount` is the number of records. |
| (Variable)      |                               | Storage for the actual string data.                                     |

The `LABL` table header is followed by an array of label subtables, one per vocabulary. Each subtable specifies label records that map glyph IDs to the associated strings. The number of vocabulary subtables is `numTables`.

The `langTagCount` and `langTagRecord` array is identical to the one used in `name` table [format 1](https://docs.microsoft.com/en-us/typography/opentype/spec/name#naming-table-format-1).

### LabelSubtable: Label vocabulary subtable

A label vocabulary subtable looks as follows:

| Type          | Name                 | Description                                               |
| ------------- | -------------------- | --------------------------------------------------------- |
| _uint16_      | `vocabularyID`       | Vocabulary ID                                             |
| _uint16_      | `languageID`         | Language ID.                                              |
| _uint16_      | `count`              | Number of label records.                                  |
| _LabelRecord_ | `labelRecord[count]` | The label records where `count` is the number of records. |

#### vocabularyID

The `vocabularyID` identifier is used in the way discussed below.

#### languageID

If a `languageID` is less than `0x8000`, it uses the same mechanism as the `platformID` 3 language identifiers in the `name` table.

If a `languageID` is equal to or greater than `0x8000`, it is associated with a language-tag record (LangTagRecord) that references a language-tag string.

For language-neutral labels, the `name` table records should use `languageID` `0x0409` (U.S. English).

### LabelRecord: Label records

| Type       | Name      | Description                                          |
| ---------- | --------- | ---------------------------------------------------- |
| _uint16_   | `glyphID` | Glyph ID                                             |
| _uint16_   | `length`  | String length (in bytes).                            |
| _Offset32_ | `offset`  | String offset from start of storage area (in bytes). |

Within one label vocabulary subtable, a `glyphID` may be used only once.

Unless there are other recommendations for a particular `vocabularyID`, it is assumed that:

- Each label string is encoded using Unicode **UTF-16BE** (_Note: this is debatable, could be **UTF-8 without BOM**_ instead).
- U.S. English or language-neutral labels should use no leading or trailing spaces, common words should be spelled in all-lowercase (while proper nouns or abbreviations using the appropriate normal-text casing), and little or no punctuation should be used.

## vocabularyID

Where the `name` records uses a `platformID`, the `LABL` table uses a `vocabularyID`.

| Value  | Description                                |
| ------ | ------------------------------------------ |
| `0`    | text contents                              |
| `1`    | private vocabulary                         |
| `2`    | development glyph names                    |
| `3`    | vendor-specific vocabulary                 |
| `4-15` | reserved                                   |
| `>15`  | registered by the specification maintainer |

### vocabularyID 0: Text contents

A label in the vocabularyID 0 defines the actual text which the labeled glyph represents, expressed as a Unicode string.

By default, vocabularyID 0 labels are language-neutral, so they use Language ID `0x0409`, however, localized labels are permissible.

If glyphs are mapped in the Unicode `cmap` table (3.1 or 3.10), label records for them are not necessary. However, text glyphs only accessible via OpenType Layout features or other such mechanisms, such as ligatures or alternate glyphs, may use labels from the vocabularyID 0. Client apps can use the vocabularyID labels to map glyphs to their text representation.

### vocabularyID 1: Private vocabulary

Labels in vocabularyID 1 can be formed freely by any font vendor, and do not need to adhere to any rules, conventions or standards. It is recommended that strings in the private vocabulary use Unicode encoding, but vendors may choose to use other means to interpret the data.

### vocabularyID 2: Development glyph names

Most font editing apps (FontLab, Glyphs, RoboFont, FontForge) allow type designers to use glyph names during font development that don’t conform with the strict [Adobe Glyph Naming](https://github.com/adobe-type-tools/agl-aglfn/) recommendations. The development glyph names are stored inside development font formats such as [UFO](http://unifiedfontobject.org/), [`.glyphs`](https://github.com/schriftgestalt/GlyphsSDK/blob/master/GlyphsFileFormat.md) or [`.vfj`](https://github.com/kateliev/vfjLib/).

Some font vendors are interested to export those “development” glyph names into fonts. Labels in the vocabularyID 2 allow them to.

### vocabularyID 3: Vendor-specific vocabulary

Labels in the vocabularyID 2 can be formed freely by any font vendor, but the assumption is made that each font vendor (as identified by the achVendID code in the OS/2 table) maintains some sort of vocabulary to which the defined labels adhere.

### Other vocabularyIDs

Labels defined for the vocabularyIDs > 15 need to be formed according to the vocabulary maintained by the registered entity. Below, I’m giving loose examples of vocabularyIDs which might be proposed for registration.

#### vocabularyID 15: [Wikipedia](https://www.wikipedia.org/)

Any label mapped to a glyph and registered in the vocabularyID 3 (Wikipedia) should be spelled exactly like the title of the Wikipedia article which corresponds to that label in the appropriate language.

#### vocabularyID 16: [The Noun Project](https://thenounproject.com/)

Any label mapped to a glyph and registered in the vocabularyID 4 (The Noun Project) should be spelled exactly like the title of the entry on The Noun Project which corresponds to that label in the appropriate language.

#### vocabularyID 17: [The Medieval Unicode Font Initiative](http://www.mufi.info/)

Labels would be formed according to the “descriptive name” used within the Medieval Unicode Font Initiative.

#### vocabularyID 18: [SIL](http://scripts.sil.org/SILPUAassignments)

Labels would be formed according to the “descriptive name” used within SIL, especially for glyphs not accessible through OpenType Layout but instead accessible through the SIL Graphite layout system.

### Optional: vocabularyID ???: Development metadata

_Note: this is debatable, and perhaps this should not be included at all._

Any record with vocabularyID must have `encodingID` 0 and `languageID` 0.

A label in this vocabularyID must be a valid JSON dictionary, encoded as UTF-8 without BOM.

It is recommended that the keys in the dictionary follow the [UFO reverse domain naming scheme](http://unifiedfontobject.org/versions/ufo3/conventions/#reverse-domain-naming-schemes).

Vendors may use labels with vocabularyID 4 to store glyph-specific metadata intended for font development. For example, the JSON data may epxress the contents of the [GLIF lib](http://unifiedfontobject.org/versions/ufo3/glyphs/glif/).

## Examples

### Example 1. “Basketball” glyph

Let’s assume that the glyph with the `glyphID` 34 represents a ball for playing basketball. In that case:

A `LABL` entry with `vocabularyID` 1 (= private vocabulary) and `languageID` `0x0409` could map `glyphID` 34 to the string with the contents `basketball`, conforming with the “all lowercase” recommendation for general labels.

Another `LABL` entry with `vocabularyID` 15 (= Wikipedia) and `languageID` `0x0409` could map `glyphID` 34 to a string with the contents `Basketball (ball)`, because that is the title of the English Wikipedia article describing the ball for playing basketball: `https://en.wikipedia.org/wiki/Basketball_(ball)`

Another `LABL` entry with `vocabularyID` 15 (= Wikipedia) and `languageID` `0x0407` (German) could map the glyph to a string `Basketball (Sportgerät)`, as this is the title of the corresponding German Wikipedia article: `https://de.wikipedia.org/wiki/Basketball_(Sportger%C3%A4t)`

Finally, an additional `LABL` entry with `vocabularyID` 16 (= The Noun Project) could map the `glyphID` 34 to the string `Basketball`, which is the title of the English entry on The Noun Project: `https://thenounproject.com/noun/basketball/`

### Example 2: “AcmeCo” logotype

If the glyph with `glyphID` 36 contains the logotype which represents the word `AcmeCo`, then that glyph may be accessible through the OpenType Layout feature `liga` or `dlig` as a ligature of the glyphs `/A/c/m/e/C/o`.

A `LABL` entry with `vocabularyID` 0 (= text contents) and `languageID` `0x0409` could exist that maps this glyph to the string `AcmeCo`.

## Discussion

In the above proposal, some portions (such as the concept of vocabularyIDs, and in particular the registered vocabularyIDs and the “text contents” vocabularyID 0), are optional, and could be “done away with” if the community deems it too complex. I’m kind of keen on at least keeping two vocabularyIDs: 0 for “text contents” and 1 for “private labels”.

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

> Humans like words better than they like numbers.

Computers, of course, like numbers better than they like words. But typography, digital text -- is actually _primarily_ a tool for _communications between humans_ these days. It’s far less about computers than some software engineers might think.

Now that `SVG` is part of OpenType, it’s more likely than ever that fonts will be used as an efficient storage for non-textual symbols. A font is not _just_ a collection of symbols. It’s a collection of symbols that, through layout systems, establishes logical and spatial relationships between these symbols. Inside a font, you can define what should happen if certain symbols occur in a sequence, under what circumstances different variants are used, and what is the spacing behavior of these symbols in relation to each other. That’s something you won’t easily implement in a cross-platform way if you just have a “bag of loose SVG graphics”. Also, fonts have (and will even more in future) a mechanism for choosing size-specific variants of the same symbol, so you won’t have to rely on linear scaling, which often produces optically sub-par results.

We have at least three layout systems on the market (OpenType Layout, AAT, SIL Graphite), and none of them provides a fully adequate mechanism to provide explanatory metadata to all kinds of glyph variants the font may have.

Glyph labeling is very useful for the textual context as well. Imagine a simple situation: you have a font which has two variants of the asterisk (`*`): one with six arms and another with five arms. You can encode one as a stylistic alternate of the other, but there isn’t really an easy way to provide the user with the information that the one glyph is “asterisk with six arms” and the other is “asterisk with five arms”.

The same goes for other kinds of specially-formed glyph variants, where the designer might want to embed some useful information about what particular glyphs are useful for, what’s their stylistic treatment etc.

And we have the “lookup by name” issue. That’s where the vocabularyID idea (which was suggested by Laurence) comes in: any entity could establish a Vocabulary and register it. Then, they could publish their vocabulary and the corresponding expected symbols. For example, an organization of map
rendering services could agree on a vocabulary for symbols used on maps. Then, font vendors could develop various fonts for use on maps, and if they label their glyphs using the mapping Vocabulary, the notion of switching the style of a map would be simple — the glyphs in a font could be looked up
using the mapping Vocabulary labels, and then, when the font is switched, a different set of symbols could be used. This could would be much more sensible to implement than doing all kinds of “corporate use of the PUA” kinds of hackery (which still could be done of course).

And finally — how the heck are you currently supposed to find a glyph which shows, say, a banana on MyFonts? There are fonts on MyFonts which have a glyph that shows a banana, I’m sure. But there is no way for the user to discover them now.

Of course font developers could make accompanying documents which describe to the user what each symbol is. But there is no standardized way to make such documents, and such documents do not travel with the font. The idea behind the `LABL` proposal is to embed this metadata inside of the font, so it doesn’t get “lost” on its travels.

Another example is mathematical typesetting: regardless of whether a typesetting engine uses the Microsoft MATH table or some TeX typesetting technique or yet another way — I think it’d be useful if the mathematical players in the field (e.g. STIX) created a vocabulary for glyphs used in math
typesetting, and embedded them into their fonts. Other mathematical font vendors could follow that. This would aid switching fonts, providing better fallback scenarios or even just developing new math fonts (because the labels would be helpful for other font developers to understand the nature of a particular glyph).

There is a large number of legitimate usage scenarios for fonts where labels are better than numbers. Either to look up glyphs by label, or to display the label to the user.

### Earlier proposals, `Zapf` table

Apple maintains a spec for the SFNT [`Zapf`](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6Zapf.html) table that partially has similar goals. I have reviewed the `Zapf` table spec before writing my own proposal. I always liked the name of the `Zapf` table and its general intent — but at the same time, I have always found the actual table format hard to understand.

There are multiple different structures (GlyphInfo, KindName, GroupInfo, GroupInfoGroup, NamedGroup, FeatureInfo), and I have a hard time imagining a sensible user interface for it. Also, the `Zapf` FeatureInfo structure seems to be closely tied to AAT. So, `Zapf` tries to expose metadata “for everything”: glyphs, glyph groups (classes) and typographic features (tied to AAT). By doing this, I think it shoots over its own goal a bit.

The `Zapf` table is almost 20 years old, and has a number of concepts which certainly are out of date (hardcoded four types of names: Apple name, Adobe name, AFII name, Unicode name). The table has been defined in pre-webfont, and even to some extent pre-web times.

With the recent discussion of color font formats, I’ve heard one recurring word of praise for the Microsoft proposal (`COLR`/`CPAL`): that it was _simple and lightweight_, therefore easy to implement. I have noticed that this aspect of the Microsoft proposal was almost universally praised, and this was what has motivated me to attempt a similar path with the `LABL` proposal. I have a feeling that the `Zapf` table is a tad too ambitious, and a bit
“over-engineered”, and therefore never really found wide adoption.

So, when writing the `LABL` proposal, I tried to learn from the lack of adoption of `Zapf` and from the overall warm welcome of the `COLR/CPAL` proposal. I tried to make the structure clean and simple to use.

I tried to create a structure that is a bit modular — e.g. the concept of registered vocabularyIDs is sort of optional. If the community finds that aspect too much of a complication, it can be removed (leaving only one or two hardcoded Encoding IDs) without invalidating the entire structure. Analogically, if the community decides to drop that idea now but revisits it in future, the notion of adding an extra Encoding ID is easy and won’t break previous implementations.

Plus, the Vocabulary concept allows for a clean separation: each vocabulary exists in a separate LABL subtable, therefore a font developer can easily set up, say, 30 labels within one particular vocabulary, and then 150 labels within another vocabulary, independently of each other. This is something I learned to like about `cmap` — that each `cmap` subtable can really be handled separately. These days, only the cmap 3.x is of relevance, but the fact that you can “safely” add or remove 0.x, 1.x or 4.x cmap subtables without interfering with the 3.x subtable, and that these subtables could address different subsets of the glyphset, always was appealing to me.

The fact that pretty much every font editing tool already has a `name` table editor, and that all platforms have the ability to parse the `name` table makes the `LABL` proposal a rather low-hanging fruit.

Since the `LABL` table itself is really trivial in structure in itself, and re-uses `cmap` structures, also means that adding support for it will be very easy. For example, I imagine that adding `LABL` support to something like fontTools/TTX is a matter of 15 minutes work.

The only administrative overhead resulting from my proposal will be the maintenance of the list of registered Vocabulary IDs. But I believe it’s unlikely we’ll ever get more than a dozen of those, so it’s still quite “cheap”.

Of course, the maintenance of the actual vocabularies or “policing” semantic conformance of the labels to a particular vocabulary is completely out of scope of the spec — just like it is out of scope of the spec to ensure that, in a particular font, the glyph with the Unicode U+0041 really depicts an uppercase “A”.

Regards,

> Adam Twardoch
