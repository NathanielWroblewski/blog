---
layout: post
title:  Start Your Day with Spotlight
---

Everyday I come into work, I do the same thing:

  * I open three tabs in Chrome: PivotalTracker, Gmail, and Github
  * I open five tabs in iTerm: one for mongoDB, one for [zeus](https://github.com/burke/zeus), one for my server, one for a rails console, and one for git
  * I open my editor to the root of my rails app

It's routine, so I wanted to automate it.  To do so, I wrote a pretty simple shell script:

```sh
#!/bin/bash

# open tabs in Chrome
open -a Google\ Chrome http://www.gmail.com
open -a Google\ Chrome https://www.pivotaltracker.com/n/workspace
open -a Google\ Chrome https://github.com/darbysmart/rails

# open iTerm2 from default terminal
open -a iTerm

# method to open tabs in iTerm and execute some command
launch () {
/usr/bin/osascript <<-EOF
tell application "iTerm"
    make new terminal
    tell the current terminal
        activate current session
        launch session "Default Session"
        tell the last session
            write text "{$1}"
        end tell
    end tell
end tell
EOF
}

# open tabs and execute code in each tab
launch "vg;cd rails;mongod"
launch "vg;cd rails;zeus start"
launch "vg;cd rails;zeus s"
launch "vg;cd rails;zeus c"
launch "vg;cd rails;subl ."
```

The only problem was that in order to run this script I'd have to open my terminal
and run the script from there.  That's not cool.

To solve this, I saved the file with a `.command` suffix and made the file executable:

```sh
$ chmod a+x good_morning.command
```
*Note the above makes the file executable for all users/groups.*

And now I use Spotlight on my Mac to find it and execute it!  So, when I sit down in
the morning, I can just type `cmd+Space` to open Spotlight, type `good morning`,
hit enter, and my workstation is automatically set up for me!

Hacky, but awesome.
