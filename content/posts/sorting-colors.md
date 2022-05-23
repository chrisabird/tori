---
layout: postXS
title: Sorting colors
date: 2013-10-06 00:00:00 +0000
category: Programming
tags: lisp, clojure, colors
---
Previously i talked about [finding similar colors](/posts/finding-similar-colors),  I'd now like to move onto sorting. 

## given a sparse set of colors how do i arrange them in an aesthetically pleasing order.
 
There is an inherent problem with this statement through. I have two axis in which I can work on screen x and y, but color is three dimensional. So we have to work out a way to minimise one of those axis or combine two to achieve something close to a pleasant lay down. 

So here is a jumble of colors from around the color red...

{{< rawhtml >}}
<div style="margin-bottom:32px;overflow:auto;">
	<div style="float:left;width:40px;height:40px;background-color:#DB9099;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F3D1C9;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EFD9D0;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F1E5DC;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D79380;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#842328;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#B77774;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AA3F55;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D29A98;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E2C3BF;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EEDED7;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#712A29;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F1A7A2;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AF5142;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AC568E;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E7D9E3;">&nbsp;</div>
</div>
<div style="clear:left;"></div>
{{</ rawhtml >}}

### What happens if we order by RGB

Sorting the colors like this...

{{< highlight clojure >}}
	(defn sort-by-rgb [colors]
	  (sort-by (juxt :rgbR :rgbG :rgbB) colors))

	(fact "can sort colors by rgb"
		(first (sort-by-rgb red-colors)) 
		   => (contains {:rgbR 113 :rgbG 42 :rgbB 41})
	  (last (sort-by-rgb red-colors)) 
	     => (contains {:rgbR 243 :rgbG 209 :rgbB 201})) 
{{</ highlight >}}

Which looks like this...

{{< rawhtml >}}
<div style="margin-bottom:32px;overflow:auto;">
	<div style="float:left;width:40px;height:40px;background-color:#712A29;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#842328;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AA3F55;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AC568E;text-align:center;">A</div>
	<div style="float:left;width:40px;height:40px;background-color:#AF5142;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#B77774;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D29A98;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D79380;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#DB9099;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E2C3BF;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E7D9E3;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EEDED7;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EFD9D0;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F1A7A2;text-align:center;">B</div>
	<div style="float:left;width:40px;height:40px;background-color:#F1E5DC;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F3D1C9;">&nbsp;</div>
</div>
<div style="clear:left;"></div>
{{</ rawhtml >}}

This doesn't look right... 

 1. Color (A) is appearing in the middle of the reds, as we're not taking into account the other colors that are being mixed in (aka the hue).
 2. Color (B) is appearing in the middle of lighter/less-intense colors. Because with RGB we're only sorting by how much of each color is present not how bright the overall color is.  

There are a few possible ways to make this look better...

### Ordering by Hue, Luminosity and Chroma. 

This method works well if you have a large set of colors or if you have a set of colors that span a large portion of the color spectrum.

For this we're going to have to switch color models again. At this point you could use [HSV](https://en.wikipedia.org/wiki/HSL_and_HSV) as it breaks color into Hue, Saturation and Brightness. But as I was working primarily with CIELab I opted for [CIELch](https://en.wikipedia.org/wiki/CIELUV_color_space#Cylindrical_representation) which can easily be converted to and has similar Hue, Chroma and Luminosity axis to work with. 

So now let's convert CIELab to CIELch and sort by Hue then Luminosity then Chroma

{{< highlight clojure >}}
	(defn cielab-to-cielch [color] 
		(let [l (:cieL color)
		     a (:cieA color)
		     b (:cieB color)
		     h (Math/atan2 a b)
		     c (Math/sqrt (+ (Math/pow a 2) (Math/pow b 2)))]
		 (if (> h 0) 
		   (merge color {:cieC c :cieH (* (/ h (Math/PI)) 180)}) 
		   (merge color {:cieC c :cieH (- 360 (* (/ (Math/abs h) (Math/PI)) 180))}))))

	;; Sorting by CIELch
	(defn sort-by-cielch [colors]
		(sort-by (juxt :cieH :cieL :cieC) (map #(cielab-to-cielch %) colors)))

	(fact "can convert from cielab to cielch"
	 (cielab-to-cielch {:cieL 88.0459 :cieA 6.5440 :cieB -3.1817}) 
		   => (contains {:cieC 7.2764792922126835 :cieH 115.92907185324125})) 

	(fact "can sort colors by cielch"
		(first (sort-by-cielch red-colors)) 
				=> (contains {:cieC 5.744894008793722 :cieH 30.45499414831985})
		(last (sort-by-cielch red-colors)) 
				=> (contains {:cieC 8.175087286105267 :cieH 121.03086930350263})) 

{{</ highlight >}}

Which looks like this...

{{< rawhtml >}}
<div style="margin-bottom:32px;overflow:auto;">
	<div style="float:left;width:40px;height:40px;background-color:#F1E5DC;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EFD9D0;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EEDED7;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D79380;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AF5142;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F3D1C9;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#712A29;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#842328;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E2C3BF;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F1A7A2;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#B77774;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D29A98;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AA3F55;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#DB9099;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AC568E;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E7D9E3;">&nbsp;</div>
</div>
<div style="clear:left;"></div>
{{</ rawhtml >}}

That's a bit better...

 1. The colors are now starting at the Orange-Red colors, moving into the Strong-Red colors and then into the Red-Blue (Purple) colors. 
 2. Within each group of hue they are now graduating from dark to light.

We're left with a small problem though. The value for Hue has a high precision, I found that limiting the hue to an integer ensured that values like (for example) 63.445 and 63.346 were both treated as 63 and then are ordered together by luminosity and chroma.

### Sorting only by Luminosity and Chroma

If you are working in a limited hue range already like the set of colors we're working with. Then you could order them by only the Luminosity and Chroma.

{{< highlight clojure >}}
	(defn sort-by-cielch [colors]
		(sort-by (juxt :cieL :cieC) (map #(cielab-to-cielch %) colors)))
{{</ highlight >}}

Which looks like this...

{{< rawhtml >}}
<div style="margin-bottom:32px;overflow:auto;">
	<div style="float:left;width:40px;height:40px;background-color:#712A29;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#842328;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AA3F55;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AF5142;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#AC568E;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#B77774;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D79380;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#DB9099;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#D29A98;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F1A7A2;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E2C3BF;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F3D1C9;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#E7D9E3;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EEDED7;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#EFD9D0;">&nbsp;</div>
	<div style="float:left;width:40px;height:40px;background-color:#F1E5DC;">&nbsp;</div>
</div>
<div style="clear:left;"></div>
{{</ rawhtml >}}

### There appears to be no correct answer
For me, I was trying to place colors into a grid that a user could select from. Ordering by Hue/Luminosity/Chroma made more sense as it allowed the person to find a lighter or darker version of the color which is a lot harder to achieve if you just order by Luminosity/Chroma.

Hopefully this serves as a quick intro for anyone looking to do something similar. If anyone else has done this with a greater level of success id be thrilled to hear from you. 