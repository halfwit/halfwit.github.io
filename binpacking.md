---
layout: page
title: "Bin Packing"
permalink: /binpacking
---

# Bin Packing

### ptpb links died, and I haven't had time to re-record. Sorry in advance! 

## It Has Been One Year Since My Last WM Issue

So for quite some time, I was going through window managers quite frequently. I'd trial one, enjoy many points of it, and eventually find another that suited for a different reason. I really liked [dwm](https://dwm.suckless.org/) for a few reasons. It had a wonderfully responsive stacking layout, and the tagging feature. I also really liked [i3](https://i3wm.org/) for the menu system, and per-window hooks. (these may not be unique, nor originated by these wm's; but this is where I encountered them first)
Along came the [wmutils](https://github.com/wmutils) project, which is basically a set of binaries for interacting with windows and the X server from the command line. It is similar in design to xdotool - but not monolithic - which I had come to appreciate in userland tools for ease with which I could learn them. Simple manpages, fewer flags, just easy to learn in one sitting.
That was the last straw, I'd just make my own wm. With blackjack.

## Tags Are Pretty Nice

Tags, or groups as some call them are a feature that has been around for a while. The essence of tags is pretty simple. When you have a window, you assign it to a particular group, or tag. If that window is in a tag with other windows, you can bring them into view or out all at once. That's not at all different from a workspace though, and what sets tags apart is that you can have more than one tag in the view at a time. This is hard to show in a video without voiceover, which I currently don't have access to; but I'll attempt it:
[demo](https://ptpb.pw/HSVe.mkv) The big win is, when you're coding, you can pull up your email client alongside your editor, write your email, and hide your email client fluidly. Try it! It's great.

## Cool, Tags Sound Neat. How Get?

I had decided to go with wmutils, but they had a cludgy, file-backed grouping mechanism. It worked mostly ok, but I was never very satisfied. There was lots of state wonkiness, and I'd end up with false information about which tags were active, which windows were in which tag, etc.
I was mulling over many ideas, from using a long running process to track variables, to using the same file-backed grouping as the others. Around the same time, I was working on [x11fs](https://github.com/sdhand/x11fs) to attempt to implement a clipboard - and was working with the X11 ATOM system. Putting two, and two, and two together, I was able to realize that I could store the tag data as a user atom. 
Thus were born the following:
 - [wmgroup](https://github.com/halfwit/wmgroup) 
 - [watom](https://github.com/halfwit/watom)
These were knocked off actually fairly fast, about a day for each, and worked as follows: 

```
# grp will set an arbitrarily named group to the window designated by wid
grp <name> <wid>

# lsgrp will list all wid's currently in the named group 
lsgrp <name>
```

Leveraging [sxhkd](https://github.com/baskerville/sxhkd) to intercept keystrokes, I can set groups for windows, and list all windows in a group.

```
# List all windows in a group, and use wmutils to toggle their visible state
lsgrp 1 | xargs mapw -t
```

And that was really enough to do a full tagging system. 

## So Why Not Tiling

Towards the end of my tenure of using a tiling window manager, I found myself doing something kinda pointless. I was opening up empty terminals, to pad out my other terminals to a particular size. Now, this seems insane, but I was mildly obsessing over [line lengths](https://baymard.com/blog/line-length-readability). 
Eventually, I was just sick of it all. I moved over to having all windows float, hand laying them out to what I soon recognized to be a predictable arrangement, regardless of which tags I had in. Some weeks of research ensued, and I eventually stumbled across the binpacking algorithms, specifically the [2D variant](https://en.wikipedia.org/wiki/Bin_packing_problem). The 2D one wasn't a perfect match, as it depends on rectangles which have a set size, and I wanted my windows to grow and fit the space as much as possible, when possible.
This would allow me, I figured, to arrange all of my windows to fit my container - which in this case is my screen - and do so with set sizes for each window. Terminals, ~68 chars wide. Browser? Normally about 80 chars, with the ability to grow/shrink on command, same with my editor. Finally, I wanted everything to center on my screen, because the eyes tend to be drawn to the center in my experience, and we may as well leverage that.
You're probably shouting suggestions at your screen, asking why I'm not using $thing. Well, I didn't know about $thing, where were you when I started this? Damnit. Where were you. Regardless, I wasn't able to find something that fit perfectly with all of my constraints, and this was born [binpack](https://github.com/halfwit/binpack). 

## Ok Then, That Was A Lot Of Reading. Binpack!

tl;dr

```
read windows from standard in minx/miny/maxx/maxy/window id
sort windows, biggest to smallest
place a window on screen, top left
bisect remaining space into two rectangles
sort rectangles
put next biggest window in smallest rectangle
bisect remaining space into rectangles
remove overlapping rectangles
sort rectangles
...
if fails, reduce size of windows

increase size of windows, repeat above algorithm
if increase fails, store last good values
center all windows on screen using last good values
write xywh + window id to stdout for all windows
```

Now, it's slightly more complicated than that, and there's also a few shortcuts in the code for instances where all windows fit at a maximum size. The whole thing was about a week of work, and it is [currently imperfect](https://ptpb.pw/zmbA.mkv), but it's good enough for a scrappy implementation. In the coming months, I plan on making a switch to accept number of monitors, and modify the logic when that occurs:
 -s 2: split all windows evenly between two groups, and perform a bin pack on each group. The resulting blobs will be centered evenly on two monitors
 -s 3: run a normal bin-packing on our windows, any excess screen will be sorted into two groups, on which a bin pack will be ran. The resulting blobs will be centered evenly on the outer two monitors, with the initial blob in the center. There is a short-cut here in that our worst case will be running a resize.

Now, armed with a fancy new algorithm, I needed a way to feed in the window sizes that I wanted in a consistant, and measured way. 

## Wmutils And Hwwm

By this point in time, I was already using wmutils exclusively. Simple hotkey movement around, coupled with simple hotkey resize was enough to make due for my window management needs, while I worked through the remaining struggles associated with implementing what I wanted.
The main idea was, list all windows, pipe them into a utility to pull the name of the window out, and use that name and awk to read out a preset from a file, which would be properly formatted for input to binpack, which would crunch the data and output the raw geometry for each window. Then using wmv all of our windows would be exactly where we wanted them to be. The first working attempt at this looked pretty darn good, and I even managed to grab a [video](https://www.youtube.com/watch?v=MSIjqTgtj2c).
A rather unforseen stumbling point was how to handle referring to open windows in X11 by their name. There's two notable atoms that carry nomenclature information in X11, WM_CLASS and WM_NAME. The latter was what I targetted initially, only much later referring to the former for what I was doing. There exists a remarkable inconsistency of what to attribute for WM_NAME, and I found myself patching various software - Surf's title string was an easy fix, [st](https://github.com/halfwit/dotfiles/blob/master/zsh/.zshrc#L117)'s was a bit more bothersome - and wrapping most programs that allowed it with a flag for the title, or convoluted xprop schemes that would fire some milliseconds after creation of a new window- it was hacky. 
Eventually, I had the notion to add gaps, which was knocked off in about 30 minutes, resulting in what you see [here](https://www.youtube.com/watch?v=cHCjnZ-6NZ8). One point to note in the video are that it is runtime-modifiable, all but the very central loop, which grabs all window events, and writes them into certain things like an auto-tagger.
Eventually, the level of hackiness using WM_NAME came to a head, and I decided to rewrite, using WM_CLASS instead.
 - [wshuf](https://github.com/halfwit/hwwm/blob/master/wshuf) is the algorithm that feeds binpack, and is called to shuffle the windows after open/close/move events
 - [hwwm](https://github.com/halfwit/hwwm/blob/master/hwwm) is the main loop of the program, and the only piece of the code that isn't runtime modifiable.
 - [window](https://github.com/halfwit/dotfiles/blob/master/x11/window),[size](https://github.com/halfwit/dotfiles/blob/master/x11/size), and [tags](https://github.com/halfwit/dotfiles/blob/master/x11/tags) constitute my personal settings, and contain some insight on to how certain features are achieved.

## In Closing, Your Honour

I'd like to thank you for taking the time to read this. The entire endeavour was a lot of work, and considering it started with trying scratch a barely annoying itch that was having to do anything to manage windows, one must ask: was it worth it? Yes, it was a great learning experience, and I've been nothing but happy with my window management since the switch to WM_CLASS. All in all, this has improved my productivity, and drastically improved my "flow state" fluidity, while all but obliterating the overhead in my window management from the tiling days. I honestly can't wait to get this up and running on the multihead setup, and I hope some of you out there are inspired. 
If you would like to try this window manager, but run in to snags, or have any questions, I can be found on Freenode, on the #hwwm channel, and others. Additionally, you can always email me at the address listed below. 
