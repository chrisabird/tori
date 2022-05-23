---
layout: post
title: Finding similar colors
date: 2013-09-28 00:00:00 +0000
category: Programming
tags: lisp, clojure, colors
---
Every once in a while I find a subject I get obsessed with. At the moment it's color, so for the next few posts I want to document some of the things I've been playing with. Starting at the beginning...

## given any x,y point in a picture find the closest color from a limited set  

Introducing plastic bird, our subject of study. 

![plastic bird](/img/plastic-bird.jpg "Fig 1")

I want to extract the color from image at position x:0 and y:0. Quick look in Gimp tells me the color is <em style="font-weight:bold;color:#c8a1a4">#c8a1a4</em>. So I can write a quick bit of code to load that image our and grab the color in its [sRGB](https://en.wikipedia.org/wiki/SRGB) component values.

{{< highlight clojure >}}
	(defn get-rgb-of-xy-in-image [x y img-path]
	  (let [img (ImageIO/read (File. img-path))
	        rgb (.getRGB img x y)]
	   {:r (bit-and (bit-shift-right rgb 16) 255) 
	    :g (bit-and (bit-shift-right rgb 8) 255) 
	    :b (bit-and rgb 255)}))

	(fact "can get a rgb color from a xy in an image"
		(get-rgb-of-xy-in-image 0 0 "resources/plastic-bird.jpg") => {:r 200 :g 161 :b 164})
{{</ highlight >}}

I now need a set of color to match again. When I was working on this problem I was presented with a set of around 2000 colors in the [cieLab](https://en.wikipedia.org/wiki/CIELAB) color space. A little research suggest that cieLab could offer better representation of the colors and more accuracy when matching (opinions on this welcome as I can no longer find the material that informed this choice).

To do the color matching I need to take the sRGB value and convert that into the CIE master space XYZ, then into LAB (this is covered in more detail in the linked Wikipedia entry for cieLab). So first sRGB to cieXyz...

{{< highlight clojure >}}
	(defn ciexyz-adjust [component]
	  (* 100
		  (let [x (/ component 255)] 
		   (if (<= x 0.04045)
		     (/ x 12.92)
		     (Math/pow (/ (+ x 0.055) 1.055) 2.4)))))


	(defn rgb-to-ciexyz [srgb]
	 (let [r (ciexyz-adjust (:r srgb))
		     g (ciexyz-adjust (:g srgb))
		     b (ciexyz-adjust (:b srgb))]
		 {:x (float (+ (* 0.4124 r) (* 0.3576 g) (* 0.1805 b)))
		  :y (float (+ (* 0.2126 r) (* 0.7152 g) (* 0.0722 b)))
		  :z (float (+ (* 0.0193 r) (* 0.1192 g) (* 0.9505 b)))}))

	(fact "can convert sRGB to cieXyz"
	 (rgb-to-ciexyz {:r 200 :g 161 :b 164}) => {:x (float 43.265125) 
		                                          :y (float 40.449436)
							  :z (float 40.649162)})
{{</ highlight >}}

See the [Wikipedia article on sRGB reverse transformation](https://en.wikipedia.org/wiki/Srgb#The_reverse_transformation) for more details. Now on to cieXyz to cieLab...

{{< highlight clojure >}}
	(defn cielab-adjust [component]
		(if (> component 0.008856)
		 (Math/pow component (/ 1 3))
		 (+ (* 7.787 component) (16 / 116))))

	(defn ciexyz-to-cielab [ciexyz]
	  (let [x (cielab-adjust (/ (:x ciexyz) 95.047))
		      y (cielab-adjust (/ (:y ciexyz) 100.000))
		      z (cielab-adjust (/ (:z ciexyz) 108.883))]
	   {:l (float (if (> z 0.008856) (- (* 116 y) 16) (* 903.3 y)))
	    :a (float (* 500 (- x y)))
	    :b (float (* 200 (- y z)))}))

	(fact "can convert CIEXYZ to CIELAB"
		(ciexyz-to-cielab {:x 43.265125 :y 40.449436 :z 40.649162}) => {:l (float 69.788445) 
			                                                              :a (float 14.84633) 
				                                                            :b (float 3.900725)})
{{</ highlight >}}

See the [Wikipedia article on cieLab forward transformation](https://en.wikipedia.org/wiki/Lab_color_space#Forward_transformation) for more details.

cieLab represents the color as a combination of its Luminocity (L) and coordinates on two positive/negative axis (A & B). "A" runs from green to magenta and "B" runs from yellow to blue. It's was now fairly clear that finding close colors was a problem of finding the shortest Euclidean distance between the start color and a single color in the set. 

So finally I can use the following distance function to find a color from the set that has the shortest distance.

{{< highlight clojure >}}
	(defn distance [x y]
	(Math/sqrt
	 (+ (Math/pow (- (:l y) (:l x)) 2)
	    (Math/pow (- (:a y) (:a x)) 2)
	    (Math/pow (- (:b y) (:b x)) 2))))

	(fact "can calculate euclidian distance"
	 (distance 
	   {:l 67.467419 :a 29.989378 :b 6.301008} 
	   {:l 42.403418 :a 45.709176 :b 10.237176}) => 29.846433854198214)
{{</ highlight >}}

There are problems with this though (which turned into a common realisation with color problems). The color space is not uniform, luckily there is a formula for this called Delta-E 2000. I won't go into the implementation but I found there to be little discernible difference in the result for the sample set of data I used. If you're very concerned with accuracy though this may be something worth looking into.

In future posts I'd like to explore image quantization (reducing an image to a limited set of colors), grouping colors (given a set of colors which are red, green, etc) and finally sorting colors (taking a set of colors and putting them into aesthetically pleasing orders). 