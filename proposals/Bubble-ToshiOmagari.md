This is a proposal for a new bubble table (BUBL?) in a font.

##Overview
Traditionally, a glyph has square bounding box to determine its space around letterforms. It works mostly okay with Latin, certainly not so much in non-Latin, and this box is why we need kerning, to compensatefor the imperfection of the square spacing. Bubble is a better alternative, made of arbitrary shapes drawn by the type designer. For more details, please see [my ATypI presentation](https://www.youtube.com/watch?v=4mh7dbcP3zQ).
Thanks to Martin Hosken for discussing the idea with me.

##Processing
When a renderer finds a bubble in the glyph, it uses it as spacing info and ignores sidebearings. After spacing with bubbles, kern feature will be applied. I said above that kerning was a necessity for square model, bubble model cannot fix all problems and you will need some manual kerning too, albeit the pairs a type designer needs to make will be much fewer.

##Efficient shapes
There are several possible options of how to store bubbles in a font.
<dl>
<dt>Standard Bezier
	<dd>Easy for type designers to draw, but hard to process collision of curves.
<dt>List of numbers ([See from 7:50 of the video above](http://www.youtube.com/watch?v=4mh7dbcP3zQ&t=7m50s))
	<dd>Probably the easiest for the renderer to deal with, but loses vertical spacing functionality. Also it's difficult for a font editor to convert back and forth.
<dt>Arbitrary polygonal shapes
	<dd>Easy to draw, easy to process. It's a bit harder to draw bubbles around round shapes.
<dt>Octagonal polygon
	<dd>Polygonal shapes but line angles are stricted to multiples of 45°. This is what SIL has implemented in Graphite.
Personally I think arbitrary polygon is the way to go.

##Other considerations
- In my presentation, I introduced a way to prevent over-kerning when bubbles do not meet each other (see from [13:05 of the video above](http://www.youtube.com/watch?v=4mh7dbcP3zQ&t=18m20s)). Although I think it's a sensible limit, this is still my personal opinion and it may be worth discussing how to handle this kind of situation.
- In some cases (e.g. Arabic), you may need a control to allow bubble to overlap, prioritising bounding box. The bubble around the resulting cluster is still useful for collision avoidance. See from [18:20 of the video above](http://www.youtube.com/watch?v=4mh7dbcP3zQ&t=18m20s)
