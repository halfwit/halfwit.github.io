---
layout: post
title: "Searching And Saving With Plumber"
---

[TL;DR, NOW SHOW ME THE DAMN CODE](https://halfwit.github.io/searching.html#plumbing)

## Standing On The Shoulders Of !/bin/sh

Several years ago, after playing with dmenu a _lot_, [dsearch](https://github.com/halfwit/dsearch) was born. It was designed to behave as an omni-search bar, staying out of the way and leaving the browser out of most of the searches I did on a computer. 

It grew, it grew and grew and grew - incorporating more and more bangs (a la DuckDuckGo)
- `!yt|!pl` - searching Youtube for videos or playlists
- `!code` - search local code bases
- `!w` - search Wikimedia sites
- `!lyrics` - search for the lyrics of a currently playing song or to match a query
- `!song` - search my music library
- `!g` - search Google

I think you get the point. Anything which could be queried for went through dsearch. It popped up, and got me to my answers, largely without ever needing a browser open for the task. This allowed me to focus on a task, without a large context switch.

## Saving Private Files

dsearch became my omni-everything, eventually. There was `!rec` to start a recording, `!snap` to take a screeshot, `!next|!play|!pause`, and it was out of hand. But there was also `!save`.

`!save` was interesting. I saved things through dmenu! Appended music/videos to playlists, added RSS feeds to my RSS feed aggregator, and stored PDF files to a specific directory with launcher entries, to name some examples. It was powerful, trustworthy, and just worked. 

## But Plan9 Is Really Cool

I was using Plan9 a lot, and dearly missed dsearch while I was there. I started working on replacement binaries<sup>[1](#disclaimer)</sup> to give me at least some of the features back, mostly in Go:
- [ytcli](https://github.com/halfwit/ytcli)
- [wkcli](https://github.com/halfwit/wkcli)
- [gcli](https://github.com/halfwit/gcli)

Eventually, after making a few toy file servers I realised I could expose most of my search endpoints as files. [searchfs](https://github.com/halfwit/searchfs) was started, and `cat /n/search/youtube/myquery` would list the results of the query. 

Under the hood, searchfs would shell out to ytcli, wkcli, gcli; it was fairly circuitous. Invoking the binaries themselves was generally faster, though the eventual goal was to have it run on a very fast desktop computer, shared out on the network to make use of very low latency of a wired connection, and caching results so subsequent identical queries could be made. 

## It Always Felt Hacky.

I never actually used this, other than in testing. It just wasn't beneficial to me. Two issues were, there was no method of interface discovery. This was solved, on paper at least: `ls /n/search` would show a list of all endpoints, and `ls /n/search/someendpoint` would show all the various filters for a search. 

There was no dmenu, no easy way to call this that would invoke the search and get the heck out of my way when it was done - it wasn't close to dsearch. Cool idea though!

## Plumbing

I won't do any justice trying to explain the plumber, and [plenty](https://mostlymaths.net/2013/04/just-as-mario-using-plan9-plumber.html/) of [others](http://doc.cat-v.org/plan_9/4th_edition/papers/plumb) have [before](https://9fans.github.io/plan9port/). 

### Storage 

Plumb allows arbitrary targets to be set, and Plumber allows arbitrary listeners. Using this, I wrote [store](https://github.com/halfwit/store) which is a glorified `plumb -d store` that fetches the remote mimetype, and [storage](https://github.com/halfwit/storage). This let me do a few awesome things. Downloading a gist, given a web link is simple.

```
# These are the plumbing rules used to match this

type	is text/plain
dst     is store
data	matches	'$protocol/$urlchars'
data	matches	'https://gist.github.com/($urlchars)'
data	set	'https//gist.githubusercontent.com/$1/raw'
attr	add	'filename=/usr/halfwit/notes/gist/$1'
plumb	to	storage
plumb	client	storage
```

`$ store https://gist.github.com/halfwit/somehash`<sup>[2](#destination)</sup>

Et viola! My computer has a file named `/usr/halfwit/notes/gist/halfwit/somehash`! Similarly, rules can be written for any sort of thing I wanted to store. Classy, simple, and the only thing left is to add the `store` command to Rio (WIP).

### Searches

A small alias (or script) can be used to wrap `plumb`, such as `alias search="plumb -t query"` in your shell profile. After that, it's just rules. Preference is king here, and I lean back to my dsearch tokens

```
query='[a-zA-Z0-9_\-./:,;@ ]+'

# Any type of query you wish to handle would require an entry
type    is query
data    matches '!yt ($query)'
data    set $1
plumb   start ytcli search $1

type    is query
data    matches '!g ($query)'
data    set $1
plumb   start gcli search $1

# [...]
```

This allows many things: 
- Rewriting rules
- `search '!yt mything' | dmenu -l 10 | plumb`
- Serve up from a faster computer, to slower ones

## Future
As this gets more use, I'm certain I'll end up writing many listeners for the plumber. `storage` does the right thing for an arbitrary file, `video` comes to mind to send a video to your preferred (networked?) player. This is, in my opinion much cleaner than [this](https://github.com/halfwit/dotfiles/blob/master/dsearch/handlersrc), [this](https://github.com/halfwit/dsearch/blob/master/youtube/savepl), or adding features to [this](https://github.com/halfwit/dsearch/blob/master/dsearch).


<small><a name="disclaimer">1</a> These mostly require API keys, and it's definitely a compromise that I've made. Things exist like [Youtube-DL](https://ytdl-org.github.io/youtube-dl/index.html) and [Googler](https://github.com/jarun/googler) to be able to search these services without needing keys</small>

<small><a name="destination">2</a> This requires use of setting the destination port explicitly, so it doesn't collide with normal plumbs</small>