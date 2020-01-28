---
layout: post
title: Side Project Lessons Part 2 - Dependency Resolution
categories:
- blog
tags:
- MUD
- Open Source
- Dependency Resolution
- Lich
---

I've been working for almost a year on a set of side projects built around a [MUD](https://en.wikipedia.org/wiki/MUD) called [DragonRealms](http://www.play.net/dr/)(DR) and it's given me a chance to solve a lot of interesting problems, both technical and soft in nature. I want to use my blog as a chance to solidify and share some of my lessons learned in both areas. [Robert Herbig](https://www.rpherbig.com/about/) has been working on this project with me and you should probably read his [introductory post](http://www.rpherbig.com/2016/09/30/a-lot-can-happen-in-a-year.html) for a background on what this side project entails.

Other Posts in this series:

* [Side Project Lessons Part 1 - Building Your Castle in a Swamp](http://rcuhljr.github.io/blog/2016/10/07/side-project-lessons-part1.html)

---

## You say potato I say load('potato.rb')

Everyone loves code reuse right? Well one of the cornerstones of making code reusable and logically separate is the ability to import other code modules and files. Whether you're in Ruby, Java, C, Lua, or AutoHotKey the language you use probably supports this concept. Unless you're working in custom script framework being evaluated by Ruby.

While it's true that we could use Ruby's `load`/`require` methods they require disk access which means it can only work in Trusted scripts (you did read [my last post](http://rcuhljr.github.io/blog/2016/10/07/side-project-lessons-part1.html) right?). Since we don't want all of the scripts we run to have to be Trusted we needed a new solution. The first thing we tried was having a section at the top of each script where we called the Lich API to run the other scripts that contain resources we want access to.

As you may remember each script is running asynchronously in it's own thread. This means race conditions can crop up easily so this solution required long pauses after script invocation. We needed to be certain that the scripts had evaluated fully. This would also create many repeated pauses if you you start ScriptA which requires ScriptB and ScriptC, and meanwhile ScriptB also requires ScriptC you'd end up waiting on C to run twice in the best case scenario. It also might turn out that your pause in A wasn't long enough if ScriptC started requiring 5 other scripts instead of just 1.

## Sometimes threading actually makes things simpler

Clearly that solution wasn't going to work longterm. Our next attempt hit on something workable and a long weekend of tinkering left us with our current solution. Here's what our require statements look like in a consuming script

{% highlight ruby %}
custom_require.call(%w(common common-arcana drinfomon events spellmonitor))
{% endhighlight %}

So calling this method will block the current script until all 5 resources (and any of their children) are loaded if they don't already exist, and then resume flow immediately. Lets look at the pieces that make this work.

### Example 1
{% highlight ruby %}
def custom_require
  lambda do |script_names|
    #Code extracted to Example 1.a
  end
end
{% endhighlight %}

Here is the declaration of the `custom_require` function we used above. It returns a lambda which if you're not familiar with Ruby you can consider it an anonymous method. If we did the blocking on load inside `custom_require` the first child script that also called require would cause the whole system to deadlock since custom_require lives in just one thread. By returning the lambda we can do the blocking inside the script that called `custom_require` leaving `custom_require` unblocked for future calls. Now whats inside the lambda?

### Example 1.a
{% highlight ruby %}
before = Script.running.find_all { |s| s.name =~ /bootstrap/ }.length
force_start_script('bootstrap', script_names)
until before >= Script.running.find_all { |s| s.name =~ /bootstrap/ }.length
  pause 0.05
end
{% endhighlight %}

The bootstrap script is is our script that will deal with the actual running of the requested scripts, so we want to block on that guy, however there may be many copies of it running which need to be able to handle.

We first count up the number of instances of the script named bootstrap that are running currently. We then force start a copy of bootstrap since there's a good chance it's already running. Then we simply sleep our current thread while waiting on the number of running copies of bootstrap to come back down to what it was before we started our copy.

## Pull yourself up

It's time to discuss the three different ways required scripts might behave when run.

* Run immediately and create new Classes or Modules in the `Scripting` Object
* Run immediately and exit without changing the `Scripting` Object
* Run continually providing functionality until explicitly stopped

Being able to tell the first two cases apart leads to one of our less clean workarounds.

### Example 2
{% highlight ruby %}
class_defs = { 'equipmanager' => :EquipmentManager, 'common' => :DRC, 'etc' => }
{% endhighlight %}

I haven't found a good solution around this, but any script that runs once and binds something into the script closure needs to be listed here so that we can verify whether it's been run before or not, and so we can clean it up if we need to. This hash associates the script name with the symbol it adds to the `Scripting` object that Untrusted scripts are bound to when running.

Using those definitions we can add a clean up mode that removes all previously defined constants.

### Example 3
{% highlight ruby %}
if args.wipe_constants
  class_defs.values.each { |symb| Scripting.send(:remove_const, symb) if Scripting.constants.include?(symb) }
  exit
end
{% endhighlight %}

This will remove any Classes or Modules added by our tracked scripts, allowing them to be run again the next time a require requests them. Assuming you don't call this script in restart mode it continues on to the main body.

Lets look at the rest of bootstrap, I'll provide annotations in comments.

### Example 4
{% highlight ruby %}
# Continue looping until out of scripts to run
until scripts_to_run.empty?
  # Pop the next script from our queue
  script_to_run = scripts_to_run.shift

  # If the script is already running or it has a class definition
  # that already exists, skip it
  if Script.running?(script_to_run) ||
    (class_defs[script_to_run] && Scripting.constants.include?(class_defs[script_to_run]))
    next
  end

  # Start the script and record our initial time
  start_script(script_to_run)
  pause 0.05
  snapshot = Time.now


  if class_defs[script_to_run]
    # If this script has a class definition, wait for it to load
    pause 0.05 until Scripting.constants.include?(class_defs[script_to_run])
  else
    # Otherwise wait until the script is gone or a quarter second has passed
    until !Script.running?(script_to_run) || Time.now - snapshot > 0.25
      pause 0.05
    end
  end
end
{% endhighlight %}

## Wasn't that simple?

Now this is not to say there aren't problems to tackle or improvements left to make. It's certainly a pain having to manually maintain a list of structures defined by scripts. We could go a convention based solution where files must bind only one constant and it must related directly to the filename (hello Java!) or we could look into solutions like Javascript has for require and namespaces. There's also the issue that a quarter second is not always the appropriate time to wait for a script to be in a steady state. We had a very interesting bug where someone added a very long running activity to a classes constructor greatly delaying when the script was actually ready to be interacted with.

This was one of the key developments to help us bring our scripts closer to the Land of Fully Fledged Software Development&trade;. Being able to provide reliable file access has lead to more maintainable code and greater code reuse.