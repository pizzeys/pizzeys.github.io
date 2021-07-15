+++ 
draft = true
date = 2021-07-06T02:35:30+01:00
title = "How I Hack PHP Apps (Now)"
+++

As some of you may know, I have a thing for finding bugs in PHP
applications. Until recently, my process for this mostly consisted of reading
the code, navigating either manually or via Github's 'references' feature (I
used to occaisionally import large codebases into PHPStorm for this if it was
needed. I'm quite glad I rarely have to do that anymore) and then using either
[ripgrep](https://github.com/BurntSushi/ripgrep) or sometimes
[phpgrep](https://github.com/quasilyte/phpgrep) to assist in finding useful
stuff. And then, when I found something that looked juicy... I started adding
var\_dump() and die() all over the place to get contextual information. Yeah...
printf debugging.

Don't get me wrong, everyone occaisionally does a bit of this. If you want to
get some information out of an application as quickly as possible, there's no
quicker route. But when your task essentially is debugging, all day every day,
and it becomes the core of your workflow, it's probably time to quit. And when
doing it with PHP apps in particular, there are a few very good reasons to.

The obvious one that almost everyone who does this has run into at some point:
PHP is basically a templating system that left college. By default, everything
you print is going straight into the output buffer to be rendered. Which means
when your vulnerable code ends up in an RSS feed somewhere, or an AJAX call that
renders a JS object, or basically anywhere that isn't an HTML page, that
quick-and-easy var\_dump you added just broke the application, and you're hunting
through Chrome's network tab to find the request to see the info you wanted.

The next step usually taken here is to, instead of printing the info, write it to
a log file instead. But what if there was another way? What if we also graduated
college, and used an actual debugger? :)

I had wanted to do this for some time, but kept kicking the can down the road
because, well, changing the direction of an oil tanker is hard. But it turns out,
getting an awesome envionment for debugging PHP sensibly is *really easy*.

* Step 0: Have a local environment for running PHP.

  OK, you probably already do, if you're reading this. But, I want to make a
  suggestion: Use Windows as your development OS, and then use WSL2. You may be
  recoiling right now, and if so, I understand, keep using your Mac or Linux
  machine. But, I changed to Windows recently. And this is my setup guide.

  If you haven't tried WSL since WSL1, it is actually *fantastic*. I primarily
  use Windows now, and am in a WSL2 terminal pretty much all day for 'real work'.

  For my hacking environment, I have mod\_userdir configured to serve a directory
  from my home directory and I just install the vulnerable applications there.

* Step 1: Use VSCode as your editor.

  I am a Vim die-hard. Switching to VSCode like all the cool kids were doing was
  a no-no for me. Then I did, and it turns out it's excellent, so whoops. For this
  use, the primary reasons are: it has WSL2 integration that just works, so when
  you open a project in WSl2, it handles all of the WSL2 magic transparently,
  including starting the instance if it's not running, installing any addon files
  needed, etc. And it has a graphical debug interface, which is kind of the point
  here. I do still use Vim emulation, I don't think that will ever change.

* Step 2: Install xdebug PHP extension.

  In your WSL2 environment, make sure the xdebug php extension is installed. For
  Ubuntu based systems, this is ``apt install php-xdebug``. You then want to head
  to your ``/etc/php/7.4/mods-available/xdebug.ini`` or equivalent and set:

  ```
  xdebug.remote_enable = 1
  xdebug.remote_autostart = 1
  ```

  And then restart Apache. That's all the configuration required (I told you this was
  easier than you thought). This setup basically tells xdebug to automatically run
  on every PHP script, and try to connect to a frontend using the default settings.
  You obviously shouldn't do this on a production server, but on a development 
  server it works fine (even if the frontend isn't available at the moment, it will
  still complete the request.)

* Step 3: Install the vscode extension

* Step 4: launch.json and 'wait for debug'

* Step 5: step through and debug stuff (screenshot)
