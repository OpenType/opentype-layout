At TYPO Berlin a couple of weeks ago — following my presentation on making fonts for USE — I got questions from a number of people regarding sending scripts with existing layout support to USE rather than legacy engines.

We've spoken in the past at OTWG meetings about defining Indic3 tags to pass Devanagari etc. to USE, and I'm going to make some fonts with which to test this (I'll make these available to anyone implementing a USE engine, but not for distribution; if more open testing is desired, we could make USE-compatible versions of open source Indic fonts, but that would likely be a lot more work: my Indic fonts are pretty close to USE-compatible already, so would only need tag changes and cross-cluster contextual GSUB put into appropriate features). But there's at least some interest in having a mechanism to direct other scripts to USE, in order to take advantage of the more predictable layout behaviour and access to all OTL features. A few people asked me about passing even 'simple' scripts such as Latin to USE.

In Windows 10, Sinhala and Tibetan — which were previously shaped with their own engines — are now passed to USE, *with no change in script tag*. I flagged this in my presentation as potentially dangerous for the same reasons as passing Indic2 lookups to USE are understood to be: fonts may contain contextual lookups that presume cluster-only processing, which will then behave unexpectedly when applied across a whole run. My understanding is that MS only tested their own Sinhala and Tibetan fonts when deciding to pass these scripts to USE. I am also concerned about the inconsistency developing between among USE implementations with regard to which scripts get passed to USE and which go to other engines (Google still pass Sinhala to their Indic engine, not to USE). It is relatively easy to make a font that will work with both older Indic engines and USE if one does this intentionally, but it will also be easy to make a Sinhala font presuming USE shaping on Windows that will fail if passed to an Indic2 engine. I presume the same would be true for Tibetan.

During one conversation in Berlin, Behdad suggested an idea that I think deserves serious exploration: that instead of defining new script tags to send specific complex scripts to USE — e.g. <dev3>, <bng3> etc. — we could establish a mechanism by which any script tag with an initial uppercase letter would be passed to USE for layout. I'd like to discuss this idea — in email or at the next meeting — and get a sense of how viable it is, and what problems might arise.

I'd also like to see if we can tidy up inconsistencies in which scripts currently get passed to USE by different implementers, before this becomes a bigger problem as more implementations emerge. USE shouldn't be yet another aspect of OTL for which font developers can't make reliable predictions.