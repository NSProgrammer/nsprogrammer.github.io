---
layout: post
title:  "UIColor - why hast though forsaken me!?"
date:   2013-11-22 12:00:00 -0800
categories: jekyll update
---

{% highlight objc %}

    [UIColor colorWithRed:0.15294117647059
                    green:0.16470588235294
                     blue:0.84705882352941
                    alpha:1.0];

{% endhighlight %}

So I start this article with a snippet of code. A very common snippet in fact.
It's simply some Objective-C that creates a `UIColor` object with red, green, blue and alpha components.
Now, aside from the alpha value, how easy is it to read this code?
How easy is it for a designer to relay color in this `float` form to developers?
In fact, when has a designer ever, in the history of iOS and Mac OS X development, provided the color for use in an app as `float` values?
I don't think there's a number closer to 0 than that.
And last, how easy is it for a developer to look at this code and understand what color is being created here?
So in this post, I'm going to take a very short amount of time making everyones _color coding_ lives easier with some `UIColor` code extensions.

## As easy as RGB...A

Now looking at the above code snippet, all I can think is, "there has to be a better way!" And, of course, there is.
The easiest way to improve on the `float` based construction of a color is to think of how we describe color in a digital world.
Well, 99 times out of a hundred, we describe color (in computers) by their red, green and blue components - or better known as RGB.
Now of course, the `UIColor` describes the RGB components, but each component is treated a `float`.
However more simply, when I speak with designers, they describe colors using a 16 million color palette,
or more specifically with the values of each component being between `0` and `255`.
Let's look at what the code would have looked like if Apple had used `0-255` as the RGB component value inputs instead:

{% highlight objc %}

    [UIColor colorWithRed:39
                    green:42
                     blue:216
                    alpha:255];

{% endhighlight %}

Wow! It's like a whole new world! Not only can you easily gauge how much of each color component is being used,
you can actually remember the values and, more importantly, verbally relay them between designer and engineer!
Well, Apple did not provide us with this API, so many developers take up the practice of converting the value in their code:

{% highlight objc %}

    [UIColor colorWithRed:39.0f/255.0f
                    green:42.0f/255.0f
                     blue:216.0f/255.0f
                    alpha:255.0f/255.0f];

{% endhighlight %}

This totally works, but it took more than **90** characters of code to describe it!
That's unreasonable, I'm a programmer and inherently hate verbosity (at least in code).
So let's take this a step further and use a handy dandy C macro to make this MUCH more compact.

{% highlight objc %}

    // Declare macro
    #define COLOR_INTS(r, g, b, a) \
    [UIColor colorWithRed:(r##.0f/255.0f)   \
                    green:(g##.0f/255.0f)   \
                     blue:(b##.0f/255.0f)   \
                    alpha:(a##.0f/255.0f)];

    // Call macro
    COLOR_INTS(39, 42, 216, 255);

{% endhighlight %}

Creating a `UIColor` is now merely 29 characters of code (excluding the macro declaration of course)!
This is wizard and much better. In fact, for most this is as much as would ever be needed!
However, let's take colors a step further.
What if I wanted to describe a color as a single value and not 3 color components and an alpha component?
Well, as it turns out, colors fit beautifully into computers because we can describe them in binary - 32 bits in fact.
Now each color component is between 0 and 255, which is 256 values. Well that just happens to perfectly fit into 8 bits!
And, as we all know, 8 bits is a byte. So if each component is a byte, we have 4 bytes and therefore 32-bits.
What else is 32 bits? An `unsigned long`!

Well, as it happens, colors don't "magically" happen to fit into 32-bits -
it's actually that computers evolved over time to support more and more bits to describe their pixels on screen.
The first bitmap capable computers were the IBM PC family of computers that could support monochrome bitmaps,
that's [1-bit color](http://www.youtube.com/watch?v=MkwLAnRf3SA) - literally on or off. The first colors in computers were with 2.5 bit color that was pretty hokey.
It's really 3 bits of color, but only 2 bits indicate colors and the last bit changed the intensity of the color on the screen and was called [CGA](http://www.youtube.com/watch?v=HwvD20n1sZE).
Next in line was [EGA](http://www.youtube.com/watch?v=C6jtJUPjYkw), 4-bits of color glory giving 16 different colors to show on screen at the same time.
Computers progressed to 8-bit monochrome providing 256 different shades of grey (eat your heart out [E.L. James](http://en.wikipedia.org/wiki/Fifty_Shades_of_Grey)).
But those 8 bits could also provide, for the first time, actual ranges of color!
8-bit color was 3 bits red, 3 bits green and 2 bits blue providing computers with color in a single byte of data.
They chose blue as the deficient color because they wanted the colors to fit into 8 bits and since
the human eye has half as many blue color receptors as red and green it made sense to cripple the color we are least able to see.
This color standard was called [VGA](http://www.youtube.com/watch?v=YGSq0ob70yU). Well, if you like 8-bit color you'll love 16-bit color!
That's 4 bits for each color plus the addition of an entirely new channel...alpha!
Alpha is the component of a pixel that indicates how transparent it is; the higher the value, the less you can see through it.
And, alas, [32-bit "true" color](http://www.youtube.com/watch?v=9W4vYyeFrFc) was born with a full byte for each component, which is perfect for 32-bit processor computers.
Now colors have progressed further all the way to 64-bit color and beyond,
however the ability to even visually see more than 16 million colors is so minute that 32-bits color has become the defacto bit depth for most screens you see today.
Well, that was a fun tangent! [Back to our post](http://25.media.tumblr.com/tumblr_luyl52VOBK1r6tf8io1_r1_500.gif)!

So given that we can represent color in 32 bits, it has become common practice to express those bits in Hex.
So the color we've been dealing with would actually be the same as `0x272AD8FF`, also known as RGBA.
Alternatively, we can state the color as ARGB: `0xFF272AD8`.
In fact, this is the most common way I've ever used to communicate colors with designers.
So if the language we communicate colors with is Hex, we should be able to extend `UIColor` with a simple category to accomplish just this.
Let's first look at the interface we want to create:

{% highlight objc %}

    typedef uint32_t UIColorARGB;
    typedef uint32_t UIColorRGB;

    @interface UIColor (Extensions)

    + (UIColor *)colorWithRGBString:(NSString*)hexString;

    + (UIColor *)colorWithARGB:(UIColorARGB)color32Bits;
    + (UIColor *)colorWithRGB:(UIColorRGB)color32Bit;

    - (UIColorARGB)argbValue;
    - (UIColorRGB)rgbValue;
    - (NSString *)rgbStringValue;

    @end

{% endhighlight %}

So with our category we want to accomplish 3 things.
1) We want to be able to create a `UIColor` with hex - which fortunately can easily be expressed in Objective-C.
2) We want to be able to create a `UIColor` using an `NSString`.
3) We want to do the reverse of 1 and 2, thereby being able to retrieve a 32-bit representation of the color as well as the `NSString` representation.

{% highlight objc %}

    // Create with ARGB value
    [UIColor colorWithARGB:0xFF272AD8];

    // Create with RGB value
    [UIColor colorWithRGB:0x272AD8];

    // Create with String
    [UIColor colorWithRGBString:@"0xFF272AD8"];
    // or
    [UIColor colorWithRGBString:@"#FF272AD8"];
    // or
    [UIColor colorWithRGBString:@"FF272AD8"];
    // or any of the above 3 without the alpha
    [UIColor colorWithRGBString:@"#272AD8"];

{% endhighlight %}

Now isn't that so much easier than using `-colorWithRed:green:blue:alpha:`?
The answer is, _Yes, yes it is!_
Normally I'd go into implementation details on how to implement these methods,
but they are honestly so straightforward now that we know how to describe color that I'll just leave it as an exercise.
That plus I wrote this while waiting for my delayed flight home and am exhausted.
I won't leave you completely hanging though, as usual, my [github repo has everything implemented](https://github.com/NSProgrammer/NSProgrammer/blob/master/code/NOBUILib/UIColor%2BExtensions.m).
