# Lookup Flags Extensions

This is a proposal on how to extend the lookup LookupFlags to support more flags
than currently can fit into the unassigned flags.

## Changes


The lookuptable is extended to support multiple flags in anticipation of extra
flag needs, and a single flag is added which has meaning for Attachment lookups :

LookupTable:

Type   | Name          | Description
------ | ------------- | -----------
uint16 | LookupType    | Different enumerations for GSUB and GPOS
uint16 | LookupFlag    | Lookup qualifiers
uint16 | SubTableCount | Number of subtables in this lookup
Offset | Subtable[SubTableCount] | Array of offset to Subtables from beginning of Lookup table
uint16 | MarkFilteringSet | Only present if UserMarkFilteringSet of LookupFlag is set
uint16 | ExtraFlag     | More lookup qualifiers, only present if ExtraFlags of LookupFlag is set

LookupFlag bit enumeration:

Type   | Name                 | Description
------ | -------------------- | -----------
0x0001 | RightToLeft          | Only used for cursive attachment
0x0002 | IgnoreBaseGlyphs     | If set, skips over base glyphs
0x0004 | IgnoreLigatures      | If set, skips over ligatures
0x0008 | IgnoreMarks          | If set, skips over all combining marks
0x0010 | UserMarkFilteringSet | If set, use mark filtering
0x0060 | Reserved             | Set to zero
0x0080 | ExtraFlags           | If set, the ExtraFlag field is included and used
0xFF00 | MarkAttachmentType   | If not zero, skips over all marks of attachmant type different from specified.

ExtraFlag bit enumeration:

Type   | Name                 | Description
------ | -------------------- | -----------
0x7FFF | Reserved             | Set to zero
0x8000 | Reserved             | Set to zero. Reserved for future flags extension

## Discussion

Do we want to extend the `ExtraFlags` to 32-bits and save a level of future chaining?
We are already expecting 4 bits to be used in future proposals.
