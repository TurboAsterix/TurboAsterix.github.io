---
title: "Safari 27, a Ghost Miro Popup, and the Most Expensive Bookmark I've Ever Debugged"
date: 2026-09-26
draft: false
description: "A corrupted Safari bookmark kept opening a Miro URL at startup. Here's how I tracked it down and fixed it."
---
A Safari 27 debugging story featuring LaunchServices, SQLite, binary plists, and a single corrupted bookmark.

For the next poor soul (or AI agent) who lands here after Googling "Safari miroapp popup at startup", here's what happened.

## The symptom

Every time Safari started, about 10 seconds later I got a popup asking to open Miro.

The URL was always identical:

`miroapp://miro.com/app/board/<my_blog_identifier>/`

The funny part? Miro wasn't installed.

## What we ruled out

Naturally, we started with the obvious.

* No open Miro tabs.

* No pinned tabs.

* No tab groups.

* Website data and cache deleted.

* Safari extensions disabled.

* New Safari profile.

* Fresh Safari container.

* New browser session.

The popup survived all of it.

## The rabbit hole

The logs showed Safari itself calling `LSOpenFromURLSpec()` about 10 seconds after launch.

That led us into increasingly questionable life choices.

### LaunchServices

`lsregister` still knew about a `miroapp:` handler.

It turned out to be a zombie registration:

* Bundle ID: `com.electron.realtimeboard`

* Version: `0.11.168`

* Path: `/Volumes/Miro/Miro.app`

* Status: Volume not mounted

Apparently I'd launched an old Electron version of Miro directly from a DMG years ago.

We successfully removed the stale LaunchServices registration.

The popup remained.

### Safari internals

Next stop:

* `SafariTabs.db`

* `CloudTabs.db`

* `session_state`

* `windows.restoration_archive`

* `bplist00`

* `NSKeyedArchiver`

We exported binary plists, searched SQLite blobs, decoded restoration archives, inspected WebKit logs, and watched Safari reconstruct its session in real time.

Everything looked clean.

## The breakthrough

At some point I remembered something my lead SAP architect taught me:

> Not every problem should be solved in the debugger.

So I tried something embarrassingly simple.

I deleted the Miro bookmark.

The popup disappeared.

Then things became even stranger.

### The reproducible bug

I recreated the bookmark.

* Same URL.

* Same bookmark folder.

The popup came back.

Then I noticed something:

The bookmark existed twice.

Finally I removed every bookmark pointing to that Miro board and created a fresh bookmark in a different folder.

The popup disappeared permanently.

## What probably happened

My best guess is that Safari had an internally corrupted bookmark object or stale metadata associated with that specific bookmark entry.

Interestingly:

* the URL itself was fine,

* a freshly created bookmark worked,

* only the original bookmark (and its duplicate/folder context) triggered Safari to launch `miroapp://` during startup.

## Lessons learned

* Safari 27 can trigger `miroapp://` without Miro being installed.

* A stale LaunchServices registration may coexist with the real issue.

* `SafariTabs.db` and `restoration_archive` are fascinating, but sometimes they're innocent.

* If a bookmark behaves suspiciously, delete it completely and recreate it instead of merely moving or editing it.

And perhaps the biggest lesson:

> Before spending hours reverse-engineering WebKit, make sure your "master data" isn't haunted.

Happy debugging.
