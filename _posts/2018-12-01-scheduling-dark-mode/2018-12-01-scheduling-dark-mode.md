---
layout: post
title: "Scheduling Dark Mode"
date: 2018-12-01
---

The release of OS X Yosemite in 2014 brought with it a complete visual overhaul of the system user interface as a tentpole feature. Alongside the widely anticipated removal of many skeuomorphic elements from Mavericks, in line with the design direction iOS had gone in a year earlier, was the addition of a rather interesting (but slightly overhyped) option in System Preferences to use a dark menu bar and dock. The API for this appearance change had already been laid the previous year in the form of [`NSAppearance`](https://developer.apple.com/documentation/appkit/nsappearance), and some apps took advantage of `NSVisualEffectView`'s [`.dark`](https://developer.apple.com/documentation/appkit/nsvisualeffectview/material/dark) material to provide an appearance that matched that of the system. macOS Mojave has a significantly more extensive version of this behavior that comes "for free" as part of AppKit's Dark Mode support, but back then apps generally relied on watching for an `AppleInterfaceThemeChangedNotification` and updating their appearance to match.

One feature which was *not* provided was the ability to change the system theme to match a schedule, allowing for doing things like using Dark Mode at night while picking the light appearance during daytime. I wanted this feature enough that I ended up writing [DarkNight](https://github.com/saagarjha/DarkNight), which is sufficiently simple that I wrote it as a launch agent instead of a full-fledged application. It uses some lesser-known private APIs to in order to do its job, so I thought it might be interesting to share how it works. We can divide its functionality into two main parts: being able to set the system appearance, and making sure this is done on time.

## Setting the system appearance

While applications have always had a way to *get* the system appearance through [`NSAppearance.current`](https://developer.apple.com/documentation/appkit/nsappearance/1531945-current), the ability to programmatically *set* the system appearance for anything other than your application is unsurprisingly not provided. This makes writing software that tries to schedule these changes rather annoying. For a while there was a global user default, `_HIEnableThemeSwitchHotKey`, that allowed for switching between themes with the keyboard shortcut <kbd>^⌥⌘T</kbd> when enabled, but this was removed within a couple of years and most applications ended falling back on AppleScript anyways:

```applescript
tell application "System Events"
	tell appearance preferences
		set dark mode to not dark mode
	end tell
end tell
```

If we're trying to do this from native Swift code, like I was when writing DarkNight, having to call out to AppleScript to do something straightforward like this is pretty ugly. If we ever have to ask ourselves how to perform a simple operation that doesn't seem to be exposed in a nice way, chances are that there is private API that does exactly what we want, to service requests from system applications that need to do the same thing. This is especially true for functionality that involves multiple subsystems internally but can be simplified into a simple API. While setting the system appearance may end up talking to a many different subsystems in the OS to do things like notifying applications and setting preferences, from a client's standpoint this should really only be a single function call that we pass in the appearance we want. Keeping this in mind, an obvious choice for places to investigate for this functionality is System Preferences, which has the ability to set the system theme:

![Appearance Settings](AppearanceSettings.png)

Since System Preferences needs to do the same thing that *we* want to do, we might be able to take a look at what it is doing and replicate it ourselves. A bit of digging shows that we want to look at `/System/Library/PreferencePanes/Appearance.prefPane/`, which provides the implementation for the "General" preference pane. Let's load the binary into Hopper and search for "theme":

![Hopper Appearance](HopperAppearance.png)

`-[AppearanceShared setTheme:]` seems interesting: in the decompiled code, we can see that it calls out to the function `SLSSetAppearanceThemeLegacy`, which it's getting from `/System/Library/PrivateFrameworks/SkyLight.framework`. Since SkyLight is part of the graphics subsystem, this is probably what we're looking for. One thing that we need to keep in mind is that Hopper is guessing at function prototypes; since it has no way of knowing how many arguments the function actually takes it makes a prediction based on the surrounding context. In this particular case, the fact that the `SLSSetAppearanceThemeLegacy` is being called with `_cmd` and the argument to `setTheme:` is strange; if we think back to our "fantasy API", we'd expect the function to be something like `SLSSetAppearanceThemeLegacy(theme)`. The first argument we are passing in, based on the code here, seems very much like a `BOOL`: while it could technically be any integral type, the fact that the only values used are `0x0` and `0x1` suggests that it is a boolean type. The other two arguments, which don't fit in with our model, and don't make much sense either: there really shouldn't be any reason to pass in the current method's selector, or what's essentially a repeat of information we're already passing in as the first parameter. So we might as well assume that the actual function probably takes a `BOOL` and returns something that we can ignore–we'll just use `void` as the return type. With the prototype out of the way, let's see if we can actually call this function correctly:

```objc
#import <objc/objc.h>

void SLSSetAppearanceThemeLegacy(BOOL);

int main() {
	SLSSetAppearanceThemeLegacy(YES);
	return 0;
}
```

Make sure to pass in `-F/System/Library/PrivateFrameworks -framework SkyLight` to link against SkyLight.framework; otherwise you'll get linker errors about not being able to find `SLSSetAppearanceThemeLegacy`. You may also get something along the lines of <samp>ld: warning: text-based stub file /System/Library/PrivateFrameworks/SkyLight.framework/SkyLight.tbd and library file /System/Library/PrivateFrameworks/SkyLight.framework/SkyLight are out of sync. Falling back to library file for linking.</samp>–just ignore this; it means that the SkyLight.tbd and the SkyLight binary don't match or something like that. It isn't important to our test program.

We still don't know what the parameter means; since it is a `BOOL` it is likely that `YES` means one of Light or Dark Mode, while `NO` is for setting the other value. Fiddling with the program makes it clear that `YES` corresponds to Dark Mode, and `NO` will enable Light Mode. *(Quick aside: DarkNight actually needs to toggle the system appearance, so it needs a way of getting the appearance instead of just setting it. There's a similar function called `SLSGetAppearanceThemeLegacy`, also in SkyLight, that takes no parameters and returns a `BOOL` signifying whether the system is in Dark Mode or not).*

## Running on a schedule

Now that we have a way to set the system appearance, we need a way to do this at sunrise and sunset. At first glance it looks like we will have to do this ourselves, possibly by finding a web API that does this for us, or by using the computer's location and [a bit of math](https://en.wikipedia.org/wiki/Sunrise_equation). But there's an easier way: macOS is doing this already, because it allows Night Shift to be scheduled based on sunrise and sunset! So we can just turn on Dark Mode whenever Night Shift turns on, and go back to Light Mode when it turns off.

Night Shift is actually implemented in the private CoreBrightness framework, where it is referred to as "blue light". Possibly this was a codename, or its nickname it had before it was given an marketing name? Either way, if we run class-dump on the framework's binary, we can find two interesting methods, `-[CBBlueLightClient setStatusNotificationBlock:]` and `-[CBBlueLightClient getBlueLightStatus:]`. class-dump can't tell us what the type of the notification block is, but we can guess that it probably is a block that has a parameter that is some sort of status. `-[CBBlueLightClient getBlueLightStatus:]` takes in a pointer to a structure which I've named `status` containing the substructures `time` and `schedule`, which have been named based on their members:


```objc
struct time {
	int hour;
	int minute;
};

struct schedule {
	struct time fromTime;
	struct time toTime;
};

struct status {
	char active;
	char enabled;
	char sunSchedulePermitted;
	int mode;
	struct schedule schedule;
	unsigned long long disableFlags;
};
```

So, to get a callback every time Night Shift changes, it looks like we can do something like this (compiling with `-F/System/Library/PrivateFrameworks -framework CoreBrightness -framework Foundation`):

```objc
#import <Foundation/Foundation.h>

struct time {
	int hour;
	int minute;
};

struct schedule {
	struct time fromTime;
	struct time toTime;
};

struct status {
	char active;
	char enabled;
	char sunSchedulePermitted;
	int mode;
	struct schedule schedule;
	unsigned long long disableFlags;
};

@interface CBBlueLightClient : NSObject
- (void)setStatusNotificationBlock:(void (^)(struct status *))block;
@end

int main() {
	[[CBBlueLightClient new] setStatusNotificationBlock:^(struct status *status) {
		NSLog(@"%d\n", status->enabled);
	}];
	[[NSRunLoop mainRunLoop] run];
	return 0;
}
```

Every time Night Shift is toggled, the status should be printed out.

Combining all of what we have above, it's trivial to recreate DarkNight: just change the contents of the block to set Dark Mode instead of printing out the status of Night Shift, e.g. `SLSSetAppearanceThemeLegacy(status->enabled)`. Add [a simple launchd property list](https://github.com/saagarjha/DarkNight/blob/master/DarkNight/com.saagarjha.DarkNight.plist) and you now have a launch agent ready to go.
