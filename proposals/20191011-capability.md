# Capability Conditions

## Introduction

Shaping engines change. In addition, not all shaping engines are implemented the same way.
This has caused problems in the past where font developers have to limit their font implementations to the lowest common denominator of all the shaping engines that they expect their font to work in. Even when bugs are fixed that draws engines closer together, it is the font implementor who has the difficulty of dealing with any tail of not yet updated engines.

The OpenType specification has been very stable for a number of years in terms of capability. No new lookups have been added to the standard for a very long time. But with the proposal of a number of new lookups, the divergence between the capabilities of different shaping engines, supporting the same script, will become very evident.

There is a need, therefore, for a font to be able select lookups based on the capabilities of the shaping engine processing the font. Thankfully there is already a mechanism for changing which lookups run based on factors other than script and language. The [FeatureVariations Table](https://docs.microsoft.com/en-gb/typography/opentype/spec/chapter2#featurevariations-table) allows different lookup sets to be run based on conditions.

This proposal is to add two new formats of [Condition Table](https://docs.microsoft.com/en-gb/typography/opentype/spec/chapter2#condition-table).

## Condition Tables

### Condition Table Format 2: Lookup Capability

The condition is that the given list of lookups, in the collection of FeatureSubstitutionRecords are all supported by the engine based on their lookup type. As such the format 2 table has no other members.

Type   | Name   | Description
-------|--------|------------
uint16 | Format | Format, = 2

### Condition Table Format 3:  Greater Than Engine version

Shaping engines change behaviour and for fonts to be able to address such changes, sometimes they need to behave differently for different versions of an engine. This condition tests the engine manufacturer and version number. This condition tests that the given version member is greater than the engine version.

If the engine tag is not the same as the engine tag of the engine processing the font, then this condition fails.

Type   | Name    | Description
-------|---------|------------
uint16 | Format  | Format, = 3
uint16 | version | Version number to compare
Tag    | engine  | Engine identifier

The current set of reserved engine tags is:

Tag  | Description
-----|-------------
atyp | Apple Typography (Apple)
ctyp | CoolType (Adobe)
dwrt | DirectWrite (Microsoft)
hrfb | Harfbuzz
icul | ICU Layout

### Condition Table Format 4: Less Than or Equal Engine Version

This condition is identical to condition format 3, except that the version comparison is inverted. The engine tag test is the same and the condition succeeds if the engine has the same tag as the given tag AND the version members is less than or equal to the version of the engine processing the font.

Type   | Name    | Description
-------|---------|------------
uint16 | Format  | Format, = 4
uint16 | version | Version number to compare
Tag    | engine  | Engine identifier
