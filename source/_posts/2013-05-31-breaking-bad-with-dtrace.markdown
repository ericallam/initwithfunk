---
layout: post
title: "Breaking Bad with DTrace"
date: 2013-05-31 21:25
comments: true
categories: [DTrace, Instruments, Xcode, iOS, locationd, debugging, simulator]
published: true
---

*If you have any feedback on this article, please contact me on [twitter][18]*

I've spent an unwise amount of time recently on a problem that arose when me and my [Code School][1] colleagues were developing the challenges for our upcoming [course on MapKit][2]. All of the challenges assume that the user has given the app permission to use it's current location, and our [KIF][3] tests in the simulator wouldn't work unless we could somehow programmatically confirm this alert box:

<img src="http://i.imgur.com/te7Spyj.jpg" style="width: 294px;" />

[KIF][3] tests run in the simulator and are built to a different target, so using private API's isn't a problem (KIF wouldn't work unless it used a ton of private API's). [Jim Puls][4] from Square (the makers of KIF) was helpful enough to chime in [this thread][5] to share how they do it at Square: by [method swizzling][6] the crap out of the `CLLocationManager` class, replacing `startUpdatingLocation`, `location`, and `locationServicesEnabled` with their own versions so the user location mechanics would be wholesale replaced, thus getting around the permission alert box.

We followed their advice and ended up mocking out `CLLocationManager` (see how in [this gist][7]) and it worked well enough, but I was unsatisfied that we had to mock out the entire location machinery just to get rid of that permissions alert box. It wasn't elegant code.

{% blockquote Antoine de Saint-Exupéry http://english.stackexchange.com/questions/38837/where-does-this-translation-of-saint-exuperys-quote-on-design-come-from %}
...perfection is finally attained not when there is no longer anything to add, but when there is no longer anything to take away...
{% endblockquote %}

I decided to keep searching for a more elegant solution and I started at what I thought was the most obvious place: `UIAlertView`.

### Searching for UIAlertView

If I could find the code where the offending `UIAlertView` is actually created then I would know which code to mock out and hopefully bypass permissions. To do this, I could have swizzled some `UIAlertView` init methods or added a [symbolic breakpoint][8] in Xcode, but I had recently come across [Mark Dalymple's][9] excellent three part series on DTrace ([1][10], [2][11], and [3][12]), and this seemed like a good opportunity to use it for a worthy real-world problem.

I created a new blank iOS 6.0 project with a single `MKMapView` that wants to show the current user location (you can [⬇download the project here][13]). I built and ran the app for the simulator to verify that it brought up the alert box and to create a binary that I would later use in Instruments as a target for a custom DTrace instrument.

![location alert](http://f.cl.ly/items/0f0U0g3L0u2y1f470g1e/iOS%20Simulator%20Screen%20shot%20Jun%201,%202013%2010.52.56%20AM.png)

Instruments is powered by DTrace and by using it it made a couple of things easier, like launching an app in the simulator as it attached DTrace probes and logged the results, as well as a cleaner interface for viewing call stack information. To create custom DTrace probes in Instruments, start by opening Instruments from Xcode:

![open instruments](http://i.imgur.com/69FkXod.jpg)

And then create a new blank document:

![blank document](http://i.imgur.com/RJeaV4F.jpg)

Use `⌘B` to build a new instrument, and you should come across this screen:

![new instrument](http://i.imgur.com/9U3uHBU.jpg)

Which presents a nice interface for creating custom DTrace scripts.  To find all methods (class or instance) called on `UIAlertView`, I ended up with this:

![uialertview](http://i.imgur.com/EAj9EjF.jpg)

The part to pay attention to is this group of boxes here:

![group](http://f.cl.ly/items/0R2S250R0b1m1y1P430i/Image%202013.06.01%2011%3A10%3A53%20AM.png)

Next up I needed to the choose the target this instrument would run against

![target](http://i.imgur.com/keNunJe.jpg)

You have to pick the executable that was built for the simulator, which you can find in `~/Library/Application\ Support/iPhone\ Simulator/6.0/Applications/`. In there you might see many directories with randomly generated names:

![rando](http://i.imgur.com/NxLWCRN.jpg)

You'll have to dig through the noise to find the executable you want:

![noise](http://i.imgur.com/CeUjegf.jpg)

Now all that's left is to hit the record button and hope we can find out where that `UIAlertView` is being initialized:

<video id="eWX4nh-HFXM" class="sublime" width="640" height="360" data-uid="eWX4nh-HFXM" data-youtube-id="eWX4nh-HFXM" data-autoresize="fit" preload="none" poster="http://f.cl.ly/items/0Q3L1g1X1t2G1I0n2k1v/UIAlertViewCast1Poster.png">
</video>

As you can see, there were 3 UIAlertView method calls that all came from `[UIWindow keyWindow]`, and one from a `performSelector`, but that doesn't help us much.  Let's update that probe to record both the module (in this case the class name), and the name of the method

![adding information](http://i.imgur.com/VvkCSnX.jpg)  

Now it's a little easier to see what's going on.  It looks like `[UIWindow keyWindow]` calls private class method `+_alertWindow` on `UIAlertView`.  

![private](http://i.imgur.com/JLhoerL.jpg)

Could this be where our location permissions alert box is being created. Fortunately, our instrument's probe automatically records the stack trace information for each of the events recorded.  To view the stack trace information, we just need to select an event and open the right hand side drawer:

![stack trace](http://f.cl.ly/items/3v3h0A3R3Q2Z24330N1n/Image%202013.06.02%2010%3A25%3A53%20AM.png)

The stack trace of the `_alertWindow` calls are all the same, and lead to a dead end. They don't seem to be related to location at all but instead are triggered on normal app startup routines.  The only other hit we got, `+_initializeSafeCategory`, is related to Accessibility and not location.

This is a dead end.

Well, not exactly.  Using DTrace I was able to eliminate the possibility that the `UIAlertView` was created inside the TestingCurrentLocation process, which means it has to be created in another process.  So I knew that somehow, my TestingCurrentLocation process was communicating with another outside process which was responsible for location permissions. Apple doesn't want to give app developers the chance of getting around the location permissions box for privacy reasons, so they would want to lock it down in another process.

### The Mystery Process

So my next goal was to figure out what this mystery outside process was. I knew that my app process and the mystery process had to communicate, which led me to research the [XPC Services API][14] which "provides a lightweight mechanism for basic interprocess communication integrated with Grand Central Dispatch (GCD) and launchd" and is an integral part of [App Sandboxing][15], so it seemed like an interested place to start.  I tried logging all methods called on parts of the Objective-C API, like `NSXPCConnection`, and `NSXPCInterface`, but there was nothing there. The next most obvious thing to probe was the c API, specifically the [xpc_connection_create][16] function. Here is how I created the probe I to log all the calls to `xpc_connection_create` with the caller and the `name` of the service with which to connect:

<video id="4z7meAsOgAQ" class="sublime" width="640" height="360" data-uid="4z7meAsOgAQ" data-youtube-id="4z7meAsOgAQ" data-autoresize="fit" preload="none" poster="http://f.cl.ly/items/113t1y2T07161b3e3a3U/xpc_connection_cast_poster.png">
</video>

As you can see, probing this function turned out to bear some real fruit. It found a call to `xpc_connection_create` that was creating a connection to something named `com.apple.locationd.registration` and the call stack confirmed that this being triggered by setting the map to display the users location:

![xpc_connection_create results](http://f.cl.ly/items/3F2e1x1L2p2c2X1c3a3R/Image%202013.06.02%2011%3A33%3A03%20AM.png)

Running `sudo ps aux | grep locationd` showed that their was indeed a `locationd` process running in the simulator located at `/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator6.0.sdk/usr/libexec/locationd` (there was also one running at `/usr/libexec/locationd` which is the location daemon for Mac OS X.)

### locationd

This gave me a new process to probe for information. But what should I look for?  I knew that if a user allowed an app in the simulator to get their location, they wouldn't be asked again even if the simulator was restarted (only by resetting the simulator would the location permissions alert box come up again).  So I knew that there had to be somewhere on the filesystem where locationd was storing the permission authorizations for simulator apps. If I could find this file (most likely a preferences file), I could possibly inject authorization for an app before it ever ran, thus never displaying the alert box even on the apps first run in the simulator.

So how do I find what files the locationd process is opening and reading and writing to? Well, that's where the `syscall` DTrace provider comes in, which allows you to probe all system calls (like `open`, `read`, `write`, `chdir`, etc.).  This provider makes it really easy to find out which files a process is opening and writing to by probing the `open` and `write` functions.  For example, here is a DTrace script for printing out all the files opened by the `locationd` process:

{% codeblock locationopens.d lang:d %}
syscall::open*:entry
/execname == "locationd"/
{
   printf("locationd open %s\n", copyinstr(arg0));
}
{% endcodeblock %}

This script will trace all system calls to `open*` (`*` is a wildcard), but only on processes with the name "locationd" (the `/execname == "locationd"/` line defines a predicate for the probe). The code between the curly-braces is the action to perform on each match. Inside the curly-braces we are printing the first argument to `open`, which is the path of the file (to find out which arguments a system call takes, use `man 2 syscall` which in this case would be `man 2 open`).  We have to use `copyinstr` to copy data from the process into the kernel's address space (where DTrace runs). 

Running this with `dtrace -s locationopens.d` (make sure you have root priviledges: run `sudo -i` to start a new shell in root) and then running our TestingCurrentLocation app in the simulator results in a couple hundred lines that look like this:

```

  0    927              open_nocancel:entry locationd open /System/Library/CoreServices/SystemVersion.bundle

  0    141                       open:entry locationd open /Users/eric/Library/Application Support/iPhone Simulator/6.0/Library/Preferences/.GlobalPreferences.plist

  0    927              open_nocancel:entry locationd open /System/Library/CoreServices/SystemVersion.bundle/English.lproj

  0    927              open_nocancel:entry locationd open /System/Library/CoreServices/SystemVersion.bundle/Base.lproj

  0    141                       open:entry locationd open /System/Library/CoreServices/SystemVersion.bundle/English.lproj/SystemVersion.strings

  0    141                       open:entry locationd open /System/Library/CoreServices/SystemVersion.bundle/English.lproj/SystemVersion.stringsdict

```

We can get rid of some of the noise by running dtrace with the `-q` option, like so:

```
$ dtrace -q -s locationopens.d
...snip a bunch of lines...
locationd open /Users/eric/Library/Application Support/iPhone Simulator/6.0/Library/Preferences/com.apple.locationd.plist
locationd open /System/Library/CoreServices/ServerVersion.plist
locationd open /System/Library/CoreServices/SystemVersion.plist
locationd open /System/Library/CoreServices/SystemVersion.bundle
locationd open /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator6.0.sdk/usr/libexec
locationd open /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator6.0.sdk/usr/libexec/locationd
...snip a bunch of lines...
```

Okay that's a lot of file opens, and it would take awhile going through every single one looking for clues.  What I really want to know is: what files does locationd *write* to when a user taps the "OK" button in the location alert box, and what data does it write? For that we are going to need another DTrace script:

{% codeblock locationwrites.d lang:d %}
syscall::write*:entry
/execname == "locationd"/
{
   printf("locationd write %s\n\n%s\n\n", fds[arg0].fi_pathname, copyinstr(arg1));
}
{% endcodeblock %}

Here we are tracing all calls to `write*` and printing out the path name using the first argument (arg0) which is a file descriptor (for more on using file descriptors in DTrace, read [this tutorial][17]). We are also printing out the second argument (arg1) which is a buffer of the contents written to the file. Now if we run the script just before we confirm we want to allow this app to use our current location, we will know where locationd is writing to save this information.

<video id="X29i348ea4I" class="sublime" width="640" height="360" data-uid="X29i348ea4I" data-youtube-id="X29i348ea4I" data-autoresize="fit" preload="none" poster="http://f.cl.ly/items/2X303t1A1s2C1Z243I0J/locationdwrites_poster.png">
</video>

This results in just one trace match writing to a file at `??/locationd/clients.plist`. If we run the `locationopens.d` script again while grepping for `clients.plist` we can find the full path to this file:

```
$ dtrace -q -s locationopens.d | grep 'clients.plist'
locationd open /Users/eric/Library/Application Support/iPhone Simulator/6.0/Library/Caches/locationd/clients.plist
```

### The solution

When I opened this file for the first time I had to find someone in the office to high-five.  Here it was, the file that tells locationd, and thus apps in the simulator, whether or not the user has permitted use of location information.  I could convert the `clients.plist` file into xml using `plutil -convert xml1 clients.plist` and then open it up and see what was inside:

{% codeblock clients.plist lang:xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.codeschool.TestingCurrentLocation</key>
  <dict>
    <key>Authorized</key>
    <true/>
    <key>BundleId</key>
    <string>com.codeschool.TestingCurrentLocation</string>
    <key>Executable</key>
    <string>/Users/eric/Library/Application Support/iPhone Simulator/6.0/Applications/D2700670-4F2D-4A4B-A881-695E9812CB86/TestingCurrentLocation.app/TestingCurrentLocation</string>
    <key>LocationTimeStopped</key>
    <real>391883613.42615497</real>
    <key>Registered</key>
    <string>/Users/eric/Library/Application Support/iPhone Simulator/6.0/Applications/D2700670-4F2D-4A4B-A881-695E9812CB86/TestingCurrentLocation.app/TestingCurrentLocation</string>
    <key>Whitelisted</key>
    <false/>
  </dict>
</dict>
</plist>
{% endcodeblock %}

Property List's (.plist) are a strange format but not impossible to read. The real important bit seemed to be the `Authorized` key which was set to `<true/>`.  I tried changing that to `<false/>` and rerunning the app in the simulator and my app no longer had access to location information and also didn't bring up the location permission alert box. If I removed the entire `<dict></dict>` block representing my app, the location permission alert box would show back up the next time I ran the app in the simulator.

I now had full control over that location alert box. All I had to do was write over the `clients.plist` file with the xml already crafted in a way to give my app access to location information. For our upcoming MapKit course, we're doing it inside our executor server (which is written in ruby), kind of like this:

{% codeblock lang:ruby %}
def write_out_location_plist_hack!
  plist = %{
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.codeschool.#{@project.project_name}</key>
  <dict>
    <key>Authorized</key>
    <true/>
    <key>BundleId</key>
    <string>com.codeschool.#{@project.project_name}</string>
  </dict>
</dict>
</plist>
}.strip

  clients_plist = File.join(
    File.expand_path("~"), 
    "Library", 
    "Application Support", 
    "iPhone Simulator", 
    "6.0", 
    "Library", 
    "Caches", 
    "locationd", 
    "clients.plist"
  )

  File.open(clients_plist, "w+") do |file|
    file.puts plist
  end
end
{% endcodeblock %}

The code above runs before the app is launched in the simulator, and it works perfectly. We were able to get rid of the nasty [CLLocationManager mocking][7] and I was finally able to move on to something more productive.

But trying to solve this problem did lead me to learn a lot more about DTrace. Before, DTrace just seemed like this magical thing used by superhero programmers. Now that I've used it so solve a real problem, it's not so magical anymore, and I am already starting to think of other problems I could solve using it that I wouldn't have been able to before. I hope that by writing about how I used DTrace to solve this problem, it will lead you to try it out the next time you are stuck on something similar.  If you do, please let me know how you did by getting in touch with me on my [twitter][18].


[1]: http://www.codeschool.com
[2]: http://www.codeschool.com/paths/ios
[3]: https://github.com/square/KIF‎
[4]: https://twitter.com/puls
[5]: https://groups.google.com/d/msg/kif-framework/_MffdyvMizU/hvbM6LJO7woJ
[6]: http://darkdust.net/writings/objective-c/method-swizzling
[7]: https://gist.github.com/rubymaverick/5689235
[8]: http://developer.apple.com/library/ios/#recipes/xcode_help-breakpoint_navigator/articles/adding_a_symbolic_breakpoint.html
[9]: https://twitter.com/borkware
[10]: http://blog.bignerdranch.com/1907-hooked-on-dtrace-part-1/
[11]: http://blog.bignerdranch.com/1968-hooked-on-dtrace-part-2/
[12]: http://blog.bignerdranch.com/2031-hooked-on-dtrace-part-3/
[13]: http://courseware.codeschool.com.s3.amazonaws.com/try_ios/TestingCurrentLocation.zip
[14]: http://developer.apple.com/library/mac/#documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingXPCServices.html
[15]: http://developer.apple.com/library/mac/#documentation/Security/Conceptual/AppSandboxDesignGuide/AboutAppSandbox/AboutAppSandbox.html#//apple_ref/doc/uid/TP40011183
[16]: http://developer.apple.com/library/mac/#documentation/System/Reference/XPC_connection_header_reference/Reference/reference.html#//apple_ref/c/func/xpc_connection_create
[17]: http://milek.blogspot.com/2005/07/file-descriptors-in-dtrace.html
[18]: http://twitter.com/eallam