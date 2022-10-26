---
title: 'Writing a C extension for Ruby in 2016'
date: 2016-11-12
permalink: /posts/2016/11/writing-a-c-extension-for-ruby-in-2016/
redirect_from:
  - /writing-a-c-extension-for-ruby-in-2016
tags:
  - Sonic Pi
  - ruby
---

Now that Ruby has crested the hype cycle, settled down and taken out a mortgage, you'd expect the posts around the community to be more about big business concerns. Whilst that might be true, I'd like to row against the tide by telling you about the fun I had figuring out how to write my first C extension - [`fast_osc`](https://github.com/xavriley/fast_osc/) - and how I made Sonic Pi 10x faster in the process.

Before we get stuck into the details of that, what is a C extension and why would you want to write one in 2016?

## Why?

Two reasons - speed and leverage. If you have a need for speed and a small well defined problem then writing an extension may be the way to go. Whilst working on [Sonic Pi](http://sonic-pi.net) we have to target the resource constrained Raspberry Pi Model B+ . With a 700MHz CPU and 512Mb we often have to party like its 1999. After some recent profiling work we identified that the [Open Sound Control](http://opensoundcontrol.org/spec-1_0) message encoding/decoding was a big hotspot in terms of performance and this was causing audio dropouts as the CPU struggled to keep up with our example pieces.

### Speed

We optimised as far as we could with plain Ruby, but our implementations relied on [`Array#pack`](http://www.rubydoc.info/stdlib/core/Array%3Apack) and [`String#unpack`](http://www.rubydoc.info/stdlib/core/String%3Aunpack) to get data in and out of the right binary format. If you take a peek at the `View source` links for these at the bottom of those pages, you might be able to see why they were a bit slow. The implementations run to hundreds of lines, many of which we didn't need as OSC only uses a limited number of types.

### Leverage

[Open Sound Control](http://opensoundcontrol.org/spec-1_0) is a widely used messaging format in music applications so there are plenty of good implementations in [most languages](https://libraries.io/search?q=osc&sort=). One of the great use cases for a C extension is when you don't want to reinvent the wheel as you can use the great work of others instead.

## Before diving in...

Writing a C extension is a little bit like doing distributed computing - it's best to avoid it until you *have no other option*. I'd initially wanted to use FFI (which I'll talk more about in another post) because, frankly, I was scared of writing any C that was going into a widely used app.

The issue with FFI is that it largely depends on the library you want to wrap - if it has C macros and structs flying around all over the place then it's likely to be more work writing the FFI wrapper than just writing the C extension in the first place.

It's also worth mentioning that there are great options for writing Ruby extensions in Go, Rust, Crystal and many other languages that can expose C bindings. I didn't choose to go with those for this project because the OSC libs weren't as mature but I'd definitely consider them in future.

## `rtosc` - nice, clean C

After looking at several OSC implementations I was starting to despair - I'd never written any C before and the canonical implementation, `liblo`, made use of macros and structs that I didn't completely understand at the time. After nearly giving up I stumbled on the [`rtosc`](https://github.com/fundamental/rtosc/) library which had embedding in mind. It consisted of two files (`rtosc.h` and `rtosc.c`) and emphasised speed by pre-allocating a buffer for the message so that assignments were kept to a minimum.

In the best spirit of open source, I enquired with the author via a [GitHub issue](https://github.com/fundamental/rtosc/issues/28) and got a thoughtful, well-reasoned and polite response within a few hours - viva la internet!

This managed to convince me to try writing a C extension and I'm glad I did.

## Getting started

In terms of bootstrapping the project, I followed this excellent blog post by James Coglan - [Your first Ruby native extension](https://blog.jcoglan.com/2012/07/29/your-first-ruby-native-extension-c/) Between this and [the older blog posts by Aaron Patterson](https://tenderlovemaking.com/2009/12/18/writing-ruby-c-extensions-part-1.html) I was able to suss out the minimum project structure to add a method to a Ruby object from C.

## Open Sound Control and the `fast_osc` API

OSC is a binary protocol which looks a bit like RPC. There's a "path" which represents the name of a remote method, a tag string which identifies the type of the args and then a series of binary encoded arguments. You can think of a basic message looking something like this:

    /methodname, "sif", ["This is a string", 123, 3.14]

You can see that the type of the arguments corresponds to a tag - `s` for a string, `i` for an int and so on. For the Ruby API I wanted to hide this so that you weren't thinking about types e.g.

```
>> FastOsc.encode_single_message("/foo", ["baz", 1, 2.0])
=> "/foo\x00\x00\x00\x00,sif\x00\x00\x00\x00baz\x00\x00\x00\x00\x01@\x00\x00\x00"
```

This meant that the C extension would have to take a Ruby array, do some kind of switch case statement on the type of each element, and finally build a tag string and an output array of the encoded versions according to the OSC spec.

This is pretty much the whole challenge but it did mean I need to touch Ruby's mysterious C API for quite a few different types. You can checkout the source to see how these fit together but I'll comment on a few here:

### Ruby's C API

#### Working out the type

    switch(TYPE(current_arg))

#### Ints and Longs

    case T_FIXNUM

In the OSC spec there's a distinction between integers less than `2^32` (ints) and those larger (longs). In the source [I convert the Ruby Fixnum to the relevant one using the `FIX2INT` and `FIX2LONG` macros](https://github.com/xavriley/fast_osc/blob/master/ext/fast_osc/fast_osc_wrapper.c#L189).

#### Floats

    case T_FLOAT:

Easy enough to convert from Ruby to C using `NUM2DBL`

#### Strings and Symbols

    case T_STRING:

Again this seemed pretty straightforward using the `StringValueCStr()` method. Ruby symbols can be converted to strings using `rb_sym_to_s()`. 

#### What about Unicode?

The trouble came when we worked out that I had broken Unicode strings. Since Sonic Pi aims to be fully emoji compliant, this had to be fixed. Luckily the Tenderlove blog came to the rescue again:

```
string_arg = rb_str_new2(string_from_c);
enc = rb_enc_find_index("UTF-8");
rb_enc_associate_index(string_arg, enc);
```

This little snippet made sure that the decoded string was passed back to Ruby as `UTF-8` rather than `US-ASCII` - phew!

#### Ruby `Time` objects

OSC also has a concept of timestamps which are pretty integral to music applications. I wanted the API to transparently convert these to and from `Time` objects. First you have to check for the object type:

    case T_DATA:

This means that you have a Ruby object. From there we check specifically for time using this:

    if (CLASS_OF(current_arg) == rb_cTime)

Now `rb_cTime` is one of the many examples of why I called the Ruby C API mysterious. This was quite tricky to find out and one of the main reasons I'm writing up this log to aid future intrepid explorers.

#### OSC Timestamps (NTP)

Finally, the OSC spec encodes time in two parts - first as the number of seconds since the year `1900` with the other part for the fraction of a second passed. This is what NTP servers use and it differs from the more common "seconds since 1970" (Unix epoch) that `Time.new` and others deal with.

The method for this was a bit hairy but after working out that I needed to Google "Unix to NTP timestamp conversion" I found something close enough on Stack Overflow and adapted it to Ruby like so:

```
#define JAN_1970 2208988800.0     /* 2208988800 time from 1900 to 1970 in seconds */

uint64_t ruby_time_to_osc_timetag(VALUE rubytime) {
  uint64_t timetag;
  double floattime;
  uint32_t sec;
  uint32_t frac;

  switch(TYPE(rubytime)) {
    case T_NIL:
      timetag = 1;
      break;
    default:
      // convert Time object to ntp
      floattime = JAN_1970 + NUM2DBL(rb_funcall(rubytime, rb_intern("to_f"), 0));

      sec = floor(floattime);
      frac = (uint32_t)(fmod(floattime, 1.0) * 4294967296); // * (2 ** 32)
      // printf("\nsec: %04x\n", sec);
      // printf("\nfrac: %04x\n", frac);
      timetag = (uint64_t)((uint64_t)sec << 32 | (uint64_t)frac);
      // printf("\ntimetag: %08llx\n", timetag);
      break;
  }

  return timetag;
}
```

### Memory management

Thankfully this wasn't to complicated as the underlying `rtosc` library just expects you to allocate a buffer big enough to hold the message you want to encode. This meant keeping track of the size for each of the different types above and summing them for a given message. For strings (which vary in size) I took the *`bytesize()`* and then rounded up to the next power of two just to be safe.

## Conclusion and further reading

After a lot of Googling, head scratching and running `rake compile` I finally had something that worked but I had no idea whether it was actually any faster. Thankfully, the benchmarks came in and they were good:

```
Encoding
-------------------------------------------------
            fast_osc    909.680k (±21.4%) i/s -      4.328M
             samsosc    271.908k (±19.3%) i/s -      1.327M
             oscruby     94.678k (±14.5%) i/s -    468.968k

Decoding
-------------------------------------------------
            fast_osc      2.635M (±22.2%) i/s -     12.435M
             samsosc    264.614k (±16.1%) i/s -      1.304M
             oscruby     36.362k (±16.3%) i/s -    179.622k
```

`oscruby` here is the non-optimised library for Ruby. `samosc` is our hand rolled optimised pure Ruby version and `fast_osc` is this C extension. As you can see there was something like a 4x improvement in Encoding to OSC and a 10x improvement in Decoding over the fastest pure Ruby implementation.

This new extension shipped with Sonic Pi v2.11 last week and has been battle tested in 1000s of live coding performances without missing a beat, so I can say that it was a complete success! (it's not often I get to say that about software projects...)

The code is all available here: https://github.com/xavriley/fast_osc

In terms of links I used whilst working on this, I'm have a gist here: https://gist.github.com/xavriley/507eff0a75d4552fa56e as there are too many to cover in detail.

One that does deserve a special mention though is the [The Ruby Cross Reference](https://rxr.whitequark.org/mri/source) maintained by whitequark which offers a search over all the symbols in the Ruby C source. I found this helpful for some of the more obscure parts (like `rb_cTime`) but sadly this resource appears to be shutting down on 2016-12-31 unless it finds a new owner - any takers? If you're interested you can contact them via the banner at the top of the site.

If you've been inspired to write your own extension, or if you have any feedback on my awful C I'd love to hear from you -  I'm @xavriley on Twitter - thanks for reading.

EDIT - one of the comments on [Reddit](https://www.reddit.com/r/ruby/comments/5cmux3/writing_a_c_extension_for_ruby_in_2016/) reminded me of this awesome and very helpful ebook [The Definitive Guide to Ruby's C API](http://silverhammermba.github.io/emberb/) which I used for reference as well.

