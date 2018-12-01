---
layout: post
title: "Controlling Dark Mode"
date: 2018-12-01
---

The system appearance has gotten a lot of attention recently with the release of macOS Mojave, and for good reason: it was a pitched as a tentpole feature of this version, and probably the most noticeable one at that. However, the feature of multiple appearances isn't new: the API to handle this, in the form of [`NSAppearance`](https://developer.apple.com/documentation/appkit/nsappearance), has been around since OS X 10.9, and OS X Yosemite introduced a watered-down form of "dark mode" that darkened the dock and menu bar as well as provided `NSVisualEffectView`s with the [`.dark`](https://developer.apple.com/documentation/appkit/nsvisualeffectview/material/dark) material.

While Apple has made it easy to *get* the current appearance, and to react accordingly when it changes by observing a `NSApplication`/<wbr>`NSWindow`/<wbr>`NSPopever`/<wbr>`NSView`'s [`effectiveAppearance`](https://developer.apple.com/documentation/appkit/nsappearancecustomization/1535147-effectiveappearance), the ability to *set* the appearance for anything other than your own application is unsurprisingly absent. Such a feature is useful if you are writing a utility that enables dark mode on a schedule, as my own [DarkNight](https://github.com/saagarjha/DarkNight) tool does. Let's see if we can find some private API to do this for us.

## Finding a suitable target
If you ever ask yourself a question question along the lines of "I'd like to perform a particular rather simple operation that is a subset of something the system does, but I have no idea how", it's generally a good sign that there is a way to accomplish what you are attempting to do through a private API. Since macOS is composed of many different subsystems that need to work together, many functions are neatly compartmentalized into private frameworks so that they can be used by other programs. For a feature like changing the system appearance, there are probably dozens of different things that must occur to make the transition occur, though most would be considered implementation details of little interest to the consumer of such an API; if we step back a little and think about it, a good  interface for something like this would be a single function that takes a parameter for the appearance we want to be applied. Let's see if we can find something like this.

An obvious choice for places to investigate is System Preferences, which has the ability to set the system theme:

![Appearance Settings](AppearanceSettings.png)

A bit of digging shows that the code for this preference pane is located in <code>/System/<wbr>Library/<wbr>PreferencePanes/<wbr>Appearance.prefPane/</code>. Let's load it up into Hopper, and search for "theme":

![Hopper Appearance](HopperAppearance.png)

Hmm, we have a very suspiciously named `-[AppearanceShared setTheme:]`, which takes in a integral parameter. In it's implementation, we see that it calls out a function called `SLSSetAppearanceThemeLegacy`, which it is getting from <code>/System/<wbr>Library/<wbr>PrivateFrameworks/<wbr>SkyLight.framework</code>. It seems like we're on the right track, because SkyLight is part of the graphics subsystem. However, we still need to confirm that this is the function we want.

## Calling `SLSSetAppearanceThemeLegacy`

We have a function that seems useful, but we aren't sure how to call it. Hopper's decompilation is usually helpful, but in this case, it doesn't look quite right: it looks like the function takes three arguments. The first one is probably a `BOOL` (actually, it could really be any integral type and we wouldn't know, but since we are only seeing one and zero being passed we can assume it is `BOOL` without much harm). But the second and third parameters are strange: one is `_cmd`, which in this case is `"setTheme:"`, and the other is the conventional argument being passed in to `-[AppearanceShared setTheme:]`, despite the fact that it seems to have already been provided. This, along with our hypothetical API we came up with earlier, makes us guess that the function actually only takes one boolean parameter; the other two must have been spurious decompilation errors. Let's test this theory out by defining a prototype for our function and calling it:

```objc
#import <objc/objc.h>

void SLSSetAppearanceThemeLegacy(BOOL);

int main() {
	SLSSetAppearanceThemeLegacy(YES);
	return 0;
}
```

Compile this with <code>clang -F/<wbr>System/<wbr>Library/<wbr>PrivateFrameworks -framework SkyLight</code> and run it. If you are running your computer in Dark Mode, you won't seem much of a difference. In that case, replace the `YES` with `NO`. Once you run this with the right parameter, you should see your system appearance change. It seems like `NO` corresponds to Light Mode, while `YES` is dark. Yes, we have our appearance setting function!

## Caveats
Of course, since this is private API, if we were to ever dynamically resolve this symbol we should always check for `NULL` (especially so given that the function's name contains "legacy" in it, though it has been this way for at least a couple years). If you tried to link against SkyLight directly, as the compilation command above did, you may have gotten this warning:

```shell
$ clang -F/System/Library/PrivateFrameworks -framework SkyLight dark_mode_test.m
ld: warning: text-based stub file /System/Library/PrivateFrameworks/SkyLight.framework/SkyLight.tbd and library file /System/Library/PrivateFrameworks/SkyLight.framework/SkyLight are out of sync. Falling back to library file for linking.
```

I think this warning is supposed to tell you that your command line tools and OS don't match, or something like that, but in this case the TBD file that SkyLight ships with seems to differ from the framework binary. I'm not sure if this is intentional.

In any case, be careful. This API might stick around for another couple years, or it might disappear like the `_HIEnableThemeSwitchHotKey` default (which let you switch between the two themes with <kbd>^⌥⌘T</kbd>, in case you were wondering).
