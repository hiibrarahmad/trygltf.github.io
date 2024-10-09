---
layout: post
title: "Animation Compression: Simple Quantization"
---
Simple quantization is perhaps the simplest compression technique out there. It is used for pretty much everything, not just for character animation compression.

# How It Works

It is fundamentally very simple:

*  Take some floating point value
*  Normalize it within some range
*  Scale it to the range of a **N** bit integer
*  Round and convert to a **N** bit integer

That’s it!

For example, suppose we have some rotation component value within the range `[-PI, PI]` and we wish to represent it as a signed **16** bit integer:

*  Our value is: `0.25`
*  Our normalized value is thus: `0.25 / PI = 0.08`
*  Our scaled value is thus: `0.08 * 32767.0 = 2607.51`
*  We perform arithmetic rounding: `Floor(2607.51 + 0.5) = 2608`

Reconstruction is the reverse process and is again very simple:

*  Our normalized value becomes: `2608.0 / 32767.0 = 0.08`
*  Our de-quantized value is: `0.08 * PI = 0.25`

If we need to sample a certain time `T` for which we do not have a key (e.g. in between two existing keys), we typically linearly interpolate between the two. This is just fine because key frames are supposed to be fairly close in time and the interpolation is unlikely to introduce any visible error.

# Edge Cases

There is of course some subtlety to consider. For one, proper rounding needs to be considered. Not all [rounding modes are equivalent](http://number-none.com/product/Scalar%20Quantization/) and there are [so many](http://www.eetimes.com/document.asp?doc_id=1274485)!

The safest bet and a good default for compression purposes, is to use symmetrical arithmetic rounding. This is particularly important when quantizing to a signed integer value but if you only use unsigned integers, asymmetrical arithmetic rounding is just fine.

Another subtlety is whether to use signed integers or unsigned integers. In practice, due to the way two’s complement works and the need for us to represent the 0 value accurately, when using signed integers, the positive half and the negative half do not have the same size. For example, with **16** bits, the negative half ranges from `[-32768, 0]` while the positive half ranges from `[0, 32767]`. This asymmetry is generally undesirable and as such using unsigned integers is often favoured. The conversion is fairly simple and only requires doubling our range (`2 * PI` in the example above) and offsetting it by the signed ranged (`PI` in the example above).

*  Our normalized value is thus: `(0.25 + PI) / (2.0 * PI) = 0.54`
*  Our scaled value is thus: `0.54 * 65535.0 = 35375.05`
*  With arithmetic rounding: `Floor(35375.05 + 0.5) = 35375`

Another point to consider is: what is the maximum number of bits we can use with this implementation? A single precision floating point value can only accurately represent up to 6-9 significant digits as such the upper bound appears to be around 19 bits for an unsigned integer. Higher than this and the quantization method will need to change to an exponential scale, similar to how [depth buffers are often encoded in RGB textures](http://aras-p.info/blog/2009/07/30/encoding-floats-to-rgba-the-final/). Further research is needed on the nuances of both approaches.

# In The Wild

The main question remaining is how many bits to use for our integers. Most standalone implementations in the wild use the simple quantization to replace a purely raw floating point format. Tracks will often be encoded on a hardcoded number of bits, typically **16** which is generally enough for rotation tracks as well as range reduced translation and scale tracks.

Most implementations that don’t exclusively rely on simple quantization but use it for extra memory gains again typically use a hardcoded number of bits. Here again **16** bits is a popular choice but sometimes as low as **12** bits is used. This is common for [linear key reduction]({% post_url 2016-12-07-anim_compression_key_reduction %}) and [curve fitting]({% post_url 2016-12-10-anim_compression_curve_fitting %}). Wavelet based implementations will typically use anywhere between **8** and **16** bits per coefficient quantized and these will vary per sub-band as we will see when we cover this topic.

The resulting memory footprint and the error introduced are both a function of the number of bits used by the integer representation.

This technique is also very commonly used alongside other compression techniques. For example, the remaining keys after [linear key reduction]({% post_url 2016-12-07-anim_compression_key_reduction %}) will generally be quantized as will curve control points after [curve fitting]({% post_url 2016-12-10-anim_compression_curve_fitting %}).

# Performance

This algorithm is very fast to decompress. Only two key frames need to be decompressed and interpolated. Each individual track is trivially decompressed and interpolated. It is also terribly easy to make the data very processor cache friendly: all we need to do is sort our data by key frame, followed by sorting it by track. With such a layout, all our data for a single key frame will be contiguous and densely packed and the next key frame we will interpolate will follow as well contiguously in memory. This makes it very easy for prefetching to happen either within the hardware or manually in software.

For example, suppose we have **50** animated tracks (a reasonable number for a normal AAA character), each with **3** components, stored on 16 bits (**2** bytes) per component. We end up with `50 * 3 * 2 = 300 bytes` for a single key frame. This yields a small number of cache lines on a modern processor: `ceil(300 / 64) = 5`. Twice that amount needs to be read to sample our clip.

Due in large part to the cache friendly nature of this algorithm, it is quite possibly the fastest to decompress.

*My GDC presentation goes in further depth on this topic, its content will find its way here in due time.*

[Up next: Advanced Quantization]({% post_url 2017-03-12-anim_compression_advanced_quantization %})

[**Back to table of contents**]({% post_url 2016-10-21-anim_compression_toc %})

