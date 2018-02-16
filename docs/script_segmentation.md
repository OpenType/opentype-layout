# Script Segmentation

## Introduction
This document aims to result in a standardised algorithm for run segmentation for the purposes of shaping. It starts by examining existing algorithms. ICU is core to many of the algorithms, and in the pseudo code, the calls to ICU are simplified to aid reading.

## Existing Algorithms

We describe the algorithms using an informal python based pseudocode.

All algorithms take careful pain to ensure that the two pairs in a punctuation pair are given then same script. The mechanisms for this are not included in the following algorithm descriptions.

### LibreOffice

LibreOffice uses ICU as the main mechanism for ascertaining the script of a character.

```python
def sameScript(sc1, sc2, ch):
    return sc1 <= USCRIPT_INHERITED or sc2 <= USCRIPT_INHERITED or sc1==sc2

def runs(text):
	scriptCode = USCRIPT_COMMON
	scriptStart = 0
	for i,c in enumerat(text):
	    sc = uscript_getScript(ch)
	    if sameScript(scriptCode, sc, c):
		if (scriptCode <= USCRIPT_INHERITED and sc > USCRIPT_INHERITED):
		    scriptCode = sc
	    else:
		yield((text[scriptStart:i], scriptCode))
		scriptStart = i
		scriptCode = sc
	if scriptStart < len(text):
	    yield((text[scriptStart:], scriptCode))
```
In addition, the algorithm ensures that runs between paired punctuation are kept together. That logic is not shown here.

### Firefox

Gecko, the text layout engine for FireFox and other Mozilla based applications, uses a very similar algorithm to LibreOffice. The only real difference is in the `sameScript` function:

```python
def isClusterExtender(ch):
    cat = u_charType(ch)
    # includes U_ENCLOSING_MARK
    if U_NON_SPACING_MARK <= cat <= U_COMBINING_SPACING_MARK ||
        0x200C <= ord(ch) <= 0x200D || # ZWJ, ZWNJ
        0xFF9E <= ord(ch) <= 0xFF9F    # katakan sound marks

def sameScript(sc1, sc2, ch):
    return sc <= USCRIPT_INHERITED or \
    		sc1 == sc2 or \
    		isClusterExtender(ch) or \
    		uscript_hasScript(ch, sc1)  # script extensions of ch include sc1?
```

### WebKit

The WebKit algorithm differs only from the LIbreOffice algorithm in that it looks ahead rather than behind.

```python
def runs(text):
	startIndex = 0
	currentScript = uscript_getScript(text[0])
	for i,c in enumerate(text[1:]):
		if treatAsZeroWidthSpace(c): continue
		nextScript = uscript_getScript(c)
		if nextScript == USCRIPT_INHERITED or nextScript == USCRIPT_COMPLEX:
			continue
		elif currentScript == USCRIPT_INHERITED or currentScript == USCRIPT_COMPLEX:
			currentScript = nextScript
			continue
		elif currentScript != nextScript and not uscript_hasScript(c, currentScript):
			yield((text[startIndex:i], currentScript))
			currentScript = nextScript
			startIndex = i
	if startIndex < len(text):
		yield((text[startIndex:], currentScript))		
```	


### Blink

The algorithm in Blink is more sophisticated in that it works with the extended script list for each character and works to merge them. It looks ahead one character.

```python
def getScripts(ch):
	'''return a list of scripts for ch, with best at the front'''
	res = uscript_getScriptExtensions(ch)
	primary = uscript_getScript(ch)
	if primary == res[0]:
		pass
	elif primary != USCRIPT_INHERITED and primary != USCRIPT_COMMON and primary != USCRIPT_INVALID_CODE:
		res.insert(0, primary)
	elif primary == USCRIPT_COMMON:
		if len(res) == 1:
			res.insert(0, primary)
			return res
		for i in range(1, len(res)):
			if res[0] == USCRIPT_LATN or res[i] < res[0]:
			    (res[0], res[i]) = (res[i], res[0])
	else:
		res.append(res.pop(0))
		res.insert(0, primary)
		for in range(2, len(res)):
			if res[1] == USCRIPT_LATIN or res[i] < res[1]:
			    (res[1], res[i]) = (res[i], res[1])
	return res

class runs(object):
	def __init__(self, text):
		self.common_preferred = USCRIPT_COMMON
		self.current_set = [USCRIPT_COMMON]
		self.next_set = []
		for r in self.runs(text):
			yield(r)
			
	def fetch(self, ch):
		self.next_set = self.ahead_set
		self.ahead_set = getScripts(ch)
		if len(self.ahead_set) == 0:
			return False
		if self.ahead_set[0] == USCRIPT_INHERITED and len(self.ahead_set) > 1:
			if self.next_set[0] == USCRIPT_COMMON:
				self.next_set = ahead_set
				self.next_set.pop(0)
			self.ahead_set = [ahead_set[0]]
		return True
		
	def mergeSets():
		current_i = 0
		priority_script = self.current_set[current_i]
		current_i += 1
		if self.next_set[0] <= USCRIPT_INHERITED:
			if len(self.next_set) == 2 and \
				priority_script <= USCRIPT_INHERITED and \
				self.common_preferred  == USCRIPT_COMMON:
				self.common_preferred = self.next_set[1]
			return True
		if priority_script <= USCRIPT_INHERITED:
			self.current_set = self.next_set
			return True
		next_i = 0
		have_priority = priority_scrint in self.next_set
		if current_i == len(self.current_set):
			return have_priority
		if not have_priority:
			priority_script = self.next_set[next_i]
			next_i += 1
			have_priority = priority_script in self.current_set[current_i:]
		res = []
		if have_priority:
			res.append(priority_script)
		if next_i != len(self.next_set):
			for sc in self.current_set[current_i:]:
				if sc in self.next_set[next_i:]:
					res.append(sc)
		if len(res):
			self.current_set = res
			return True
		return False

	def resolveCurrentScript(self):
		res = self.current_set[0]
		return self.common_preferred if res == USCRIPT_COMMON else res

	def runs(self, text):
		ahead_set = getScripts(text[0])
		lasti =0
		for i,ch in enumerate(text[1:]):
			if not self.fetch(ch):
				break
			if not self.mergeSets(self):
				script = self.resolveCurrentScript(self)
				self.current_set = self.next_set
				yield((text[lasti:i], script))
				lasti = i
		script = self.resolveCurrentScript(self)
		self.current_set = [USCRIPT_COMMON]
		yield((text[lasti:], script))
```

One question is whether all this machinery is needed. This is certainly a fair question given that of the characters that have an extended script list, most of them have a default script of COMMON or INHERITED and therefore will take the script of characters around them. Many of the remaining characters with a specific default script, most of these are digits, which, apart from appropriate script based styling, are unlikely to cause problems on a run boundary.

Nearly all of the rest of the characters are combining characters or are almost certainly not going to start a run, and so will take the script of the characters preceding them.

There still remain a few characters which can cause problems. In particular `U+0BAA TAMIL LETTER PA` and `U+0BB5 TAMIL LETTER VA`. It may be that adding these characters to the COMMON script may resolve the problem.

## Issues

This section discusses various issues that need to be addressed when segmenting text into runs according to script.

### Script Boundaries

All the algorithms support inherited and common script characters taking the script of characters around them. All of them resolve the script for such characters based first on the characters that precede them. Then, only if this does not resolve the question are the characters following considered until a script is found. Thus:

```
AAACCCBBB
```

Where `A` is script A, `C` is script common and `B` is script B. This is resolved to:

```
AAAAAABBB
```

This can mean a run break is introduced between what was the `C` and the `B`.

The fact that the run breaking algorithm may miscategorise the script of a common character is not a problem unless that character undergoes specific script only styling. If the `C` characters here should be rendered/shaped differently according to whether they resolve to script `A` or `B`, then their correct categorisation becomes important.

But this is the inherent limitation of such an algorithm. Without further information to resolve the ambiguity, common characters may well end up with the wrong script on a script boundary. Inherited characters have less problem with this since they rarely occur on a script boundary, and if they do will always want to take the script of the base character that precedes them, to stay with that base character.

### Unknown Characters

Characters are added to the Unicode standard all the time, and while there is only one new Unicode version per year, users still want to use those characters as soon after standardisation as they can. In addition, it can take a long time for changes to the standard to result in improved application support on a user's machine. Therefore it would be good if it were possible to give the segmentation algorithm some risiliance towards future character additions. There are two primary approaches to this:

* Treat all unknown characters as COMMON or INHERITED.
* Have fallback script values associated with all, or a significant subset of, undefined characters. This can be done at the block level rather than codepoint level.

The advantage of the second approach is that for the blocks where the script of undefined characters is clear, the results will almost certainly be correct (or the issue will be resolved correctly with a later Unicode release). Where there is confusion over which script to use, COMMON or UNDEFINED can be used, either falling back to the first approach or to the existing one.
