---
title: 'Neutron Accelerators and Drum Machines with Sonic Pi'
date: 2015-01-22
permalink: /posts/2015/01/neutron-accelerators-and-drum-machines-with-sonic-pi/
redirect_from:
  - /neutron-accelerators-and-drum-machines-with-sonic-pi
tags:
  - Sonic Pi
---

Working with [Sonic Pi](http://sonic-pi.net) one of the exciting things about being able to code music is the potential for using algorithms. For the non-technical reader, I like to think of algorithms like following a recipe. You might start with the same ingredients but the order in which you do things can affect to outcome.

When we think about music people often understand things at an intuitive level but become uneasy about the idea of using rules or recipes to generate it. "Formulaic" is a dirty word when used to describe pop songs for example, but it doesn't have to be that way. I'm going to look at one particular algorithm, Euclid's algorithm, to create musical rhythms.

## A method for spacing out rhythms

Despite all the complex words I'm about to use, if all boils down to this: if I have a number of beats that I want to play in a given space (say 2 bars), how do I space those out as evenly as possible?

That's obviously easy with regular rhythms that divide cleanly. If I want to play 8 beats across two bars of 8th notes it would look like this:

```
x . x . x . x . | x . x . x . x .
```

That's 8 `x`'s spaced across the 16 possible beats. Easy peasy.

But what about numbers that don't divide cleanly, for example, 3 into 8?

Let's start with the solution first:

```
x . . x . . x .
```

You can see we have to use a combination of long and short notes to fill the bar evenly. The above pattern uses groups of 3's and 2's to fill the space. You could easily notate it as:

```
332
```

if you count the distances from one beat to the next. Let's look at a few more examples of these odd rhythms:

```
# 5 into 8
x . x . x x . x
22121

# 13 into 16
x . x x x x . x | x x x . x x x x
2111211121111
```

One thing you might notice is that these aren't the only way to divide up the bar. To take our 3 into 8 rhythm there are actually three possible options:

```
x . . x . . x . #== 332
x . x . . x . . #== 233
x . . x . x . . #== 323
```

The last of those options appears so often in so many kinds of music that it's often called "the mother rhythm". Everything from Cuban clave to Elvis' "Hound dog". That said, if you look closely they are all variations on the same grouping: 2 lots of 3 and one lot of 2. That means we only need to generate one version and then we can rotate it to produce the others. So how are we going to generate it then?

## Back to the future: Euclid's algorithm from 300 B.C.

It turns out that Euclid thought about this too - how do you space things out as evenly as possible? Thousands of years later a scientist named Bjorklund was facing the same problem when he was setting up a Neutron Accelerator. He needed to fire neutrons evenly in a given space of time (a bit like our rhythm problem) and, being a scientist, he came up with a clever way of doing that.

```ruby
    def distribute(accents, total_beats)
      res = []

      total_beats.times do |i|
        # makes a boolean based on the index
        # true is an accent, false is a rest
        res << ((i * accents % total_beats) < accents)
      end

	  res
    end
```

That's the algorithm represented in Ruby. You don't have to understand it as it's probably not the simplest way of explaining it, but it's there to show how concise a problem like this can be when expressed in code.

Years later a percussion-playing, scientifically-minded music professor was looking at the same problem applied to musical rhythms and came across Bjorklund's research. He took the idea and explored it in a paper ["The Euclidean Algorithm Generates Traditional Musical Rhythms" (Toussaint 2005)](http://cgm.cs.mcgill.ca/~godfried/publications/banff.pdf) from which I've taken these ideas.

## In Sonic Pi...

I've added a function to Sonic Pi called `distribute` which allows you to generate arrays that fit these evenly spaced patterns. Here's some sample code:

```ruby
def distribute(accents, total_beats, beat_rotations=0)
  res = []

  total_beats.times do |i|
    # makes a boolean based on the index
    # true is an accent, false is a rest
    res << ((i * accents % total_beats) < accents)
  end

  res.ring
end

class SonicPi::Core::RingArray
  def as_x_notation
    self.to_a.map {|x| x ? 'x' : '.'}.join(' ')
  end

  def as_beat_groups
    self.to_a.slice_before {|x| x }.to_a.map(&:count).join
  end
end

puts distribute(3, 8, 0).inspect
puts distribute(3, 8, 0).as_beat_groups
puts distribute(3, 8, 0).as_x_notation
```

If you paste in the above code to SonicPi and run it, you should see this in the output window:

```
 ├─ [true, false, false, true, false, false, true, false]
 ├─ 332
 └─ x . . x . . x .
```

From here it's not too hard to map over those arrays to turn them into musical patterns. Here's an example you can play with:

```ruby
comment do
  # this function is available in SonicPi v2.4 and upwards
  # if you're using a version with this included you can 
  # delete this distribute function, otherwise uncomment it
  # above
  def distribute(accents, total_beats, beat_rotations=0)
    res = []

    total_beats.times do |i|
      # makes a boolean based on the index
      # true is an accent, false is a rest
      res << ((i * accents % total_beats) < accents)
    end

    res.ring
  end
end

# Monkey patching is never a good idea
# just say no kids...
class SonicPi::Core::RingArray
  def as_x_notation
    self.to_a.map {|x| x ? 'x' : '.'}.join(' ')
  end

  def as_beat_groups
    self.to_a.slice_before {|x| x }.to_a.map(&:count).join
  end
end

use_bpm(120)

def play_sample_for_sequence(pattern, sample_name, sleep_time = 0.25)
  pattern.each.with_index do |beat, i|
    sample sample_name if !!beat
    sleep sleep_time
  end
end

uncomment do
  live_loop(:hh) do
    with_fx :level, amp: 1 do
      cue :heartbeat
      play_sample_for_sequence(distribute(11, 16), :drum_cymbal_closed)
    end
  end

  live_loop(:bd) do
    sync :heartbeat
    play_sample_for_sequence(distribute(5, 16), :drum_bass_hard)
  end

  live_loop(:sn) do
    with_fx :level, amp: 1 do
      cue :heartbeat
      play_sample_for_sequence(distribute(2, 16).to_a.rotate(4), :drum_snare_hard)
    end
  end

  live_loop(:bass) do
    with_fx :level, amp: 1 do
      use_synth :tb303
      cue :heartbeat
      distribute(3, 8).each do |beat|
        play scale(:a2, :minor_pentatonic).choose, release: 0.3 if beat
        sleep 0.5
      end
    end
  end
end
```

Here's a demo of me live coding with the above:

<iframe width="100%" height="166" scrolling="no" frameborder="no" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/187459917&amp;color=ff5500&amp;auto_play=false&amp;hide_related=false&amp;show_comments=true&amp;show_user=true&amp;show_reposts=false"></iframe>

## Why does this work?

Musically I think it's fair to say that last example sounds pretty cool. But why? I think that the appeal of these rhythms ("Euclidean Rhythms" as Toussaint calls them) boils down to this: because the spacing is always even, when we layer these kinds of beats on top of each other we get strong "cross rhythms". In the example above I've deliberately put the snare on beats 2 and 4 to provide a strong backbeat. All the other rhythms bounce off the strength of the snare rhythm.

I think another possible reason is that our brains enjoy patterns and order even if we can't quite tell what the pattern is. Imagine the difference between two knitted jumpers, one with a completely random colour for every stitch, the other with a strong geometric pattern. Psychologically we're more likely to prefer the geometric pattern (unless the random knit *happens* to be really cool).

Having these functions in our toolkit means that we can reach musical sounding results even faster, without having to rely on things being *totally* random but still retaining an element of surprise.

What do you think? Does it sound cool to you? Will algorithmic music ever be widely accepted? Hit me up on twitter @xavriley if you have any comments or questions.

## Further rhythming

This topic has already been covered in several other places, but not as far as I know in Sonic Pi yet. There's a very cool HTML5/Javascript version of this kind of drum machine available at [http://www.groovemechanics.com/euclid/](http://www.groovemechanics.com/euclid/) which has been on Hacker news. Also there's a patch for Max called [Polyrhythmus](https://www.youtube.com/watch?v=14tw_DRev_4) which is along the same lines. I wanted to cover the musical aspects behind the algorithm in a bit more depth for this post. There's also a good Wikipedia page on [Euclidean rhythm](https://en.wikipedia.org/wiki/Euclidean_rhythm) with links to other resources.
