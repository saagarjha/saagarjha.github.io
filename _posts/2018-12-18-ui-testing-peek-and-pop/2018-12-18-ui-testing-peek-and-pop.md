---
layout: post
title: "UI Testing Peek and Pop"
---

We have a shorter article this time, aimed at solving a specific problem: UI testing 3D Touch using [XCTest](https://developer.apple.com/documentation/xctest). [XCUIElement](https://developer.apple.com/documentation/xctest/xcuielement), representing "a UI element in an application" as Apple's documentation puts it, has quite a bit of functionality allowing for a rather broad variety of interactions, including taps, gestures, and key presses. However, one rather glaring omission is the lack of any API to perform 3D Touches, which are necessary if your application needs to test its peek and pop functionality, as [break](https://github.com/saagarjha/break) does. As usual, if there isn't any public API to do something, there's probably private API that will. Let's find it and put it to good use.

## How XCTest works (and what it means for us)

One thing to keep in mind before we start is to make sure we understand how XCTest works. When you create a UI testing target in Xcode, what you're actually building is a bundle that is loaded by the a "test runner" (which is another app, separate from the one you're testing), which then uses the Objective-C runtime to find and call every method that starts with "test" in your test suite. It is important to take into consideration the fact that our ability to communicate with the main application is limited: since we are running out-of-process, we can't execute code in our app at all, and we can't interact with views outside of the accessibility-based UI testing API we have. This immediately eliminates the possibility of messing around with UIKit to simulate a 3D Touch, or making a view think it needs to present itself in a previewing context. This leaves us with abusing XCTest itself.

## Into the innards of XCTest

Since we don't really have access to anything other than XCTest, let's crack it open and see if there's anything good inside. First, we need to locate it:

```console
$ fd XCTest.framework /Applications/Xcode.app/
/Applications/Xcode.app/Contents/Developer/Platforms/AppleTVOS.platform/Developer/Library/Frameworks/XCTest.framework
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/Frameworks/XCTest.framework
/Applications/Xcode.app/Contents/Developer/Platforms/AppleTVSimulator.platform/Developer/Library/Frameworks/XCTest.framework
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/Library/Frameworks/XCTest.framework
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/Library/Frameworks/XCTest.framework
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks/IMSharedUtilities.framework/Frameworks/XCTest.framework
```

Apart from the strange occurrence of XCTest inside of IMSharedUtilities, it seems like there is one framework for each platform that XCTest supports. Since our tests run in the iOS Simulator, let's grab the one from iPhoneSimulator.platform.

Of course, the first thing we do is look for any methods suggestive of 3D Touch, but searching for "3d" returns nothing of interest. We're going to have to get more creative with our search: presumably any relevant method would have some sort of argument to set the pressure, so let's look for "pressure" instead:

```console
$ nm /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/Library/Frameworks/XCTest.framework/XCTest | grep -i "pressure"
00000000000200e4 t +[XCPointerEvent pointerEventWithType:buttonType:coordinate:pressure:offset:]
0000000000020971 t -[XCPointerEvent pressure]
0000000000020987 t -[XCPointerEvent setPressure:]
000000000001f7e7 t -[XCPointerEventPath pressDownWithPressure:atOffset:]
000000000012d1d8 s _OBJC_IVAR_$_XCPointerEvent._pressure
```

Bingo! We have a couple of interesting methods, namely `-[XCPointerEventPath pressDownWithPressure:atOffset:]` and `-[XCPointerEvent pointerEventWithType:buttonType:coordinate:pressure:offset:]`. We don't know how to call these, but we can load the binary into Hopper and find places calling these methods, which leads us to `-[XCUIEventGenerator forcePressAtPoint:orientation:handler:]`. (It looks like [Craig isn't the only one](https://youtu.be/0qwALOOvUik?t=5688) who mixes up Force Touch and 3D Touch.)

{% include aside.html content="If you search the list of methods for \"force\", you may notice that `XCUIElement`'s `PrivateEvents` category defines an unexposed `forcePress` method. If all you need to do is simulate a 3D touch, this is enough; however, this method is not enough for simulating peek and pop because we need precise control over pressure." %}

# Simulating 3D Touch events

Let's focus on the code that looks like it's generating a 3D Touch:

![Hopper disassembly of XCTest focused on -[XCUIEventGenerator forcePressAtPoint:orientation:handler:]](HopperXCTest.png)

It appears that all we need to do is create an `XCPointerEventPath` (pressing down, then lifting up at the right spot), add it to an `XCSynthesizedEventRecord`, then tell `XCTRunnerDaemonSession.sharedSession` to synthesize the event. First, we can set up headers for the methods we need to call, basing parameter types on how they're being used:

```objc
#import <XCTest/XCTest.h>

@interface XCPointerEventPath : NSObject
- (id)initForTouchAtPoint:(struct CGPoint)point offset:(NSTimeInterval)offset;
- (void)pressDownWithPressure:(double)pressure atOffset:(NSTimeInterval)offset;
- (void)liftUpAtOffset:(NSTimeInterval)offset;
@end

@interface XCSynthesizedEventRecord : NSObject
- (id)initWithName:(NSString *)name interfaceOrientation:(UIInterfaceOrientation)orientation;
- (void)addPointerEventPath:(XCPointerEventPath *)path;
@end

@interface XCTRunnerDaemonSession : NSObject
+ (id)sharedSession;
- (void)synthesizeEvent:(XCSynthesizedEventRecord *)event completion:(void (^)(NSError *))completion;
@end
```

Now we can add our own implementation as a category on `XCUIElement`.

```objc
@interface XCUIElement (ThreeDTouch)
@property(readonly) UIInterfaceOrientation interfaceOrientation;
- (void)forcePressWithForce:(double)force duration:(NSTimeInterval)duration;
@end

@implementation XCUIElement (ThreeDTouch)
@dynamic interfaceOrientation;

- (void)forcePressWithForce:(double)force duration:(NSTimeInterval)duration {
	XCPointerEventPath *eventPath = [[XCPointerEventPath alloc] initForTouchAtPoint:[self coordinateWithNormalizedOffset:CGVectorMake(0.5, 0.5)].screenPoint offset:0];
	[eventPath pressDownWithPressure:force atOffset:0];
	[eventPath liftUpAtOffset:duration];
	XCSynthesizedEventRecord *eventRecord = [[XCSynthesizedEventRecord alloc] initWithName:@"force touch" interfaceOrientation:self.interfaceOrientation];
	[eventRecord addPointerEventPath:eventPath];
	[XCTRunnerDaemonSession.sharedSession synthesizeEvent:eventRecord
	                                           completion:^(NSError *error) {
		                                         assert(!error);
	                                           }];
}
@end
```

There are a couple things to note here. One, we are initiating the 3D Touch at the center of the UI element (that's what the offset of `CGVectorMake(0.5, 0.5)` does), guessing that this will usually end up doing the right thing. We also stick the private `XCUIElement.interfaceOrientation` property here because it's necessary to construct `eventRecord`. In addition, the level that we are generating events at precludes niceties such as automatically scrolling elements so that they're visible, so we need to make sure that the point we're trying to tap is on the screen before trying to call this method. And finally, the completion *should* be using `XCTAssertNotNil`, but unfortunately this requires `self` to be a `XCTestCase` in Objective-C (but not in Swift, which [uses `_XCTCurrentTestCase` to grab the current test case](https://github.com/apple/swift/blob/6e7051eb1e38e743a514555d09256d12d3fec750/stdlib/public/Darwin/XCTest/XCTest.swift#L57) instead of relying on `self`).

With this method, the only question left is using it to generate a peek or pop event. This seems to be the most fiddly part, but in my experience a force of `1.0 / 3.0` with a duration of a couple seconds is enough to get a stable enough peek to grab a screenshot of it, at least on the iPhone XS simulator running iOS 12.1 that ships as part of Xcode 10.1. Anything significantly harder or shorter presses ends up triggering a pop.
