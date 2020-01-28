---
layout: post
title: Side Project Lessons Part 1 - Building Your Castle in a Swamp
categories:
- blog
tags:
- MUD
- Open Source
- User Settings
- Lich
---

I've been working for almost a year on a set of side projects built around a [MUD](https://en.wikipedia.org/wiki/MUD) called [DragonRealms](http://www.play.net/dr/)(DR) and it's given me a chance to solve a lot of interesting problems, both technical and soft in nature. I want to use my blog as a chance to solidify and share some of my lessons learned in both areas. [Robert Herbig](https://www.rpherbig.com/about/) has been working on this project with me and you should probably read his [introductory post](http://www.rpherbig.com/2016/09/30/a-lot-can-happen-in-a-year.html) for a background on what this side project entails.

---

## Everyone said I was daft for building here

A good starting point seems like going over the environment of the Lich project we're working in. This should provide some insight into the unique problems facing us as we try and develop software in this environment. I won't fully go into the solutions we've found for each problem but they serve to highlight the constraints we're working in and may serve as touch points for later posts.

At the high level Lich is a scripting engine that runs as an "[aggressive proxy](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)" between the game client and the game server. It's designed to allow you to send commands to the client or game as if they came from the other. To facilitate this it supports the running of Scripts which are interpreted as native ruby code.

![Lich Diagram](/assets/LichDiagram.png)

These scripts have the use of an API provided by Lich. This API provides access for reading and writing to the data flow between game and client. The way this is accomplished might sound familiar to those who work with Javascript. Ruby allows you to create [Bindings](https://ruby-doc.org/core-2.3.1/Binding.html) which are a snapshot of the execution state at a given point. This context object can be handed into [eval](https://ruby-doc.org/core-2.3.1/Binding.html#method-i-eval) along with the contents of the scripts, giving them access to a shared evaluation context. By building this binding off of the API object scripts all gain access to the API as well. Each script eval is spun up in it's own thread to allow scripts to run concurrently. This binding behavior actually brings up the first common problem we ran into.

## Stay Out of Our Castle You Stinking English Pig Dogs

There are actually two different versions of the eval/binding that Lich can run scripts can run in. A script can either be Trusted or Untrusted (the default). Scripts running in Untrusted mode are running at [$SAFE=3](http://phrogz.net/programmingruby/taint.html#table_20.1). This is a mostly outdated system that was an attempt to allow safe sandboxing of potentially dangerous code. This setting means no network or disk access, but also limits the access to things created in this mode. Trusted scripts are running like normal full ruby scripts. This leads to a variety of _interesting_ behaviors.

In practice Trusted scripts can't see classes, modules, and methods created in an Untrusted script. Untrusted scripts can see Trusted items but can't invoke any methods that would result in the object taking an action requiring trust. This can easily lead to a lot of subtle and hard to trace bugs if a user accidentally trusts a script they shouldn't or vice versa.

Our eventual solution to this was to declare that all of our scripts should be Untrusted, except for our installer/updater script. Any script wanting to take an action requiring trust passes a message to the Trusted script and the action is handled by that other script in its own thread, upon completion it unblocks the call by the Untrusted script.

This Trusted/Untrusted issue comes back and bites us later though. You can execute code snippets manually while in the game, easily verifying the behavior of some code. However Lich runs all of these snippets as Trusted. This makes it really hard to check on the state of Untrusted objects while debugging since the snippet can't see them.

## Collaboration Crisis (No more Python, this is a Ruby post)

The Lich script ecosystem provides a script repository that allows users to upload and share scripts, however it's quite unwieldy. Scripts don't update to the latest version by default so when I fix a terrible bug many users may miss the fix for weeks or months. Only one person can upload a script so working on scripts co-operatively is exceedingly painful. In addition it becomes another place you need to remember to check in your code to and you need to remember to upload every single changed file. If people have some of your scripts auto updating but not others it can quickly break when you make dependent changes to both scripts.

In the end we developed our own repository equivalent that feeds off of a Github repo. We develop scripts normally and anyone running our repository script gets the latest versions from our repo when they first start up. This isn't a perfect solution since someone with multiple characters can update the scripts out from under another character who was already running when the second character starts, but it's a huge step forward in ease of development and behavior for the users.

## Fix Things Where They're Broken

Lich itself isn't maintained in a publicly accessible repository, nor do we have an option to contribute directly. The maintainer plays a different game so focus is often divided. Combined with the fact that there's no issue tracker means that relying on fixes to the platform isn't always viable. Due to this problems in Lich are often fixed through bad workarounds and wrappers in the scripting layer leading to duplication and more bugs.

An example is when the `move(direction)` command didn't handle several failure messages specific to DR so we ended up having to write our own movement wrapper that detects failed movements and restarts the move over and over instead of just adding the failure messages in to the method in question.

This second-class nature of DR also affects sharing existing documentation with users, because many of the resources for Lich are often only partially correct for our game. We find ourselves frequently reinventing the wheel; overlooking something in Lich.rbw is easy, it's 13k lines in one file which often looks like the following.

### Actual code that I hope I don't need to understand
{% highlight ruby %}
1.times {
  @@current_room_count = XMLData.room_count
  foggy_exits = (XMLData.room_exits_string =~ /^Obvious (?:exits|paths): obscured by a thick fog$/)
  if room = @@list.find { |r| r.title.include?(XMLData.room_title) and r.description.include?(XMLData.room_description.strip) and (r.unique_loot.nil? or (r.unique_loot.to_a - GameObj.loot.to_a.collect { |obj| obj.name }).empty?) and (foggy_exits or r.paths.include?(XMLData.room_exits_string.strip) or r.tags.include?('random-paths')) and (not r.check_location or r.location == Map.get_location) and check_peer_tag.call(r) }
    redo unless @@current_room_count == XMLData.room_count
    @@current_room_id = room.id
    return room
  else
    redo unless @@current_room_count == XMLData.room_count
    desc_regex = /#{Regexp.escape(XMLData.room_description.strip.sub(/\.+$/, '')).gsub(/\\\.(?:\\\.\\\.)?/, '|')}/
    if room = @@list.find { |r| r.title.include?(XMLData.room_title) and (foggy_exits or r.paths.include?(XMLData.room_exits_string.strip) or r.tags.include?('random-paths')) and (XMLData.room_window_disabled or r.description.any? { |desc| desc =~ desc_regex }) and (r.unique_loot.nil? or (r.unique_loot.to_a - GameObj.loot.to_a.collect { |obj| obj.name }).empty?) and (not r.check_location or r.location == Map.get_location) and check_peer_tag.call(r) }
      redo unless @@current_room_count == XMLData.room_count
      @@current_room_id = room.id
      return room
    else
      redo unless @@current_room_count == XMLData.room_count
      @@current_room_id = nil
      return nil
    end
  end
}
{% endhighlight %}

We ended up making a method that handles lag around command submission by submitting the command repeatedly until the game sends a recognized response, not realizing for several months that a very similar method existed already, undocumented in the Lich code.


## Method Missing Goes Where?

Another problematic point is that since our scripts are evaluating inside the Lich environment any odd choices it has made can have disastrous consequences. It took almost 9 months to finally trace down some truly strange behavior to code living in Lich itself.

{% highlight ruby %}
class NilClass
  def method_missing(*args)
    nil
  end
end

class TrueClass
  def method_missing(*usersave)
    true
  end
end

class FalseClass
  def method_missing(*usersave)
    nil
  end
end
{% endhighlight %}

For anyone not familiar with Ruby, the `method_missing` method is an incredibly powerful meta-programming tool. It receives any calls to the object with method names that don't exist. However with great power comes the possibility to do terrible things. Here Lich makes it so if you ever have a `nil`, `true`, or `false` value and accidentally call a method on it, nothing will fail and it will return a `nil`, `true`, or `false` value as if the method had run. This lead to hair pulling debugging since a snippet like this that would throw a runtime error normally runs without error. As an example.

### Note the Typo!
{% highlight ruby %}

# Lets make our settings class
Class Settings
  def do_something?
    true
  end
end

settings = Settings.new

# Ok lets see if we were supposed to do_something, I've forgotten
if setings.do_something? # Well we got a falsey value, I guess we shouldn't do it!
  do_some_incredibly_important_things
end
{% endhighlight %}

## Democracy is the Worst!

With all this pain, Lich is still the best option out there warts and all. More than that its open ecosystem means that if something causes us enough pain there's always a viable solution if we we're willing to take it.

We've recently forked Lich and are now maintaining our own version. This has allowed for faster improvements and fixes for DR specific issues and we've made it a simple toggle for users to move on or off our custom Lich install. This facilitated extending Lich support to other front ends so it now works with the single largest 3rd party front end for DR. This gives us unparalleled reach, we're over 100 active users at any given time (about 20% of the player base). In addition I've added features like an untrusted equivalent to the in game snippet evaluator to facilitate debugging and testing.


