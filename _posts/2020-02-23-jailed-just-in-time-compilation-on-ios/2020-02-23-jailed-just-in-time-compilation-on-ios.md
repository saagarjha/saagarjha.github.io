---
layout: post
title: "Jailed Just-in-Time Compilation on iOS"
---

Just-in-time compilation on iOS normally requires applications to possess the dynamic-codesigning entitlement, a privilege that Apple uniquely awards to system processes that require the high-performance tiers of JavaScriptCore. "True" just-in-time compilers require the ability to generate executable pages with an invalid code signature, a practice that is usually prohibited on iOS for third-party apps because it sidesteps code validation guarantees that Apple would like to enforce. While these applications cannot use `mmap`'s `MAP_JIT` without this entitlement (the usual way to create a RWX region for JIT purposes), there is a method that does work on devices without a jailbreak, though its combination of being unfit for the App Store and really only being useful for speeding up virtual machines makes it seemingly unknown outside of the emulation community. The technique relies on a somewhat arcane side effect of how debugging works on iOS to enable a slightly more limited JIT.

{% include aside.html type="Update" date="6/24/20" content="Preliminary testing on iOS 14 seems to indicate that Apple has changed the kernel so that this trick no longer works." %}

## Introducing the W^X JIT

The simplest way to implement a JIT is to create pages that have both `PROT_WRITE` and `PROT_EXEC` (along with `PROT_READ`–this isn't the [bulletproof JIT](https://www.blackhat.com/docs/us-16/materials/us-16-Krstic.pdf)) enabled simultaneously, writing code into this region, and executing it. As this code is generated on the fly, it lack a code signature to back it, and `mmap` (or its Mach VM equivalents) will only allow these kinds of mappings if the process requesting them possesses the dynamic-codesigning entitlement and passes the `MAP_JIT` flag as mentioned previously. But we don't actually need both permissions at the same time: unless we're generating self-modifying code, we only need the write permissions when writing the code to memory and the execute permissions when executing it. In fact, if we just continually flipped the permissions of the pages back and forth between `PROT_WRITE` and `PROT_EXEC` based on when we were generating or running code, we'd be able to implement a just-in-time compiler while still maintaining the exclusivity of [W^X](https://en.wikipedia.org/wiki/W%5EX)–we'd never have a page be both at once. Many platforms other than iOS enforce this policy by default as a rudimentary security mitigation, including OpenBSD.

While this approach works, continuously changing page permissions is often quite slow. A better solution for performance is to (ab)use memory mappings to map the same physical page twice, with two virtual addresses, one of which is accessible with write permissions and one which enables execute permissions. From the perspective of virtual memory, the address space is still W^X, but by using the appropriate pointer to access the memory the region is effectively RWX.

## The `CS_DEBUGGED` loophole

On iOS there is normally no reason for a third-party process to need to possess invalid pages except for one: when it is being debugged. Since setting breakpoints requires overwriting code with an appropriate trapping instruction, debugging a process [must disable](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/bsd/kern/kern_cs.c#L217) the `CS_KILL` and `CS_HARD` flags that would ordinary cause a process to be killed when its code signature becomes invalid; a program in this state instead has the `CS_DEBUGGED` flag [set on it](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/bsd/kern/kern_cs.c#L220).

The usual way this flag gets set is using Xcode to debug the app, which causes debugserver to attach the process by [using `ptrace` with the `PT_ATTACHEXC` request](https://github.com/llvm/llvm-project/blob/bae33a7c5a1f220671e6d99cda21749afe2501a6/lldb/tools/debugserver/source/MacOSX/MachProcess.mm#L2552) (which the same as the deprecated `PT_ATTACH`, except it causes signals to be delivered as Mach exceptions; see below). However, relying on debugserver to make our JIT work is somewhat inconvenient and cumbersome: ideally there would be a way to do this without having to be connected to Xcode all the time. Since [we cannot attach to ourselves](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/bsd/kern/mach_process.c#L484), it'd be nice if we could create a new, temporary process with the sole purpose to attach to ours to set `CS_DEBUGGED`…except that this is nonjailbroken iOS, where we can't spawn new processes. Hmm.

A closer look at `ptrace`'s documentation reveals an interesting request: `PT_TRACE_ME`, intended to be used by a process that expects to be traced. In addition to the interesting property that it is called by the *child* process (that is: ours, not the debugger's!), it *also* [disables code signing validation](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/bsd/kern/mach_process.c#L191)!

{% include aside.html type="Fun fact" content="Interestingly, it disables validation in the *parent* process as well for some reason. I wonder if calling this in an ordinary process will end up disabling validation in its parent process, launchd, or if this is blocked by MAC." %}

So all we need to do is call `ptrace` with the `PT_TRACE_ME` request (the other arguments are ignored) and we'll have all we need to implement a W^X JIT (unfortunately, a true RWX JIT would still require dynamic-codesigning, because `mmap` checks for the entitlement specifically when granting `MAP_JIT` requests). While the `<sys/ptrace.h>` isn't the iOS SDK, the function is still present and loaded into every process. In C we can just forward declare the function and the appropriate constants and dynamic linking will take care of the rest:

```c
#include <sys/types.h>

#define PT_TRACE_ME 0
int ptrace(int, pid_t, caddr_t, int);

int main(void) {
	ptrace(PT_TRACE_ME, 0, NULL, 0);
}
```

In Swift, the process is a little more involved, but still fairly straightforward:

```swift
import Darwin

let PT_TRACE_ME: CInt = 0
let ptrace = unsafeBitCast(dlsym(dlopen(nil, RTLD_LAZY), "ptrace"), to: (@convention(c) (CInt, pid_t, caddr_t?, CInt) -> CInt).self)

ptrace(PT_TRACE_ME, 0, nil, 0)
```

## Limitations

This isn't a RWX JIT (which isn't a huge deal as you can still map the memory twice), but there are other limitations to consider. Since this is ARM, [the normal cache-flushing recommendations apply](https://github.com/WebKit/webkit/blob/7e6418e6296a1abe0ce4406838c08f2a95dfeb29/Source/JavaScriptCore/assembler/ARM64Assembler.h#L2811). Unlike processes with dynamic-codesigning, which get access to ["jumbo VA spaces"](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/bsd/kern/kern_exec.c#L3061), iOS applications can normally only allocate a limited amount of virtual memory (which is determined using [a fairly elaborate calculation](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/osfmk/arm/pmap.c#L9868) based on the size of physical memory).

However, one major issue is actually described in description for `PT_TRACE_ME` in [the man page for `ptrace(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/ptrace.2.html):

> PT_TRACE_ME
> This request is one of two used by the traced process; it declares that the process expects to be traced by its parent.  All the other arguments are ignored.  **(If the parent process does not expect to trace the child, it will probably be rather confused by the results; once the traced process stops, it cannot be made to continue except via ptrace().)**  When a process has used this request and calls execve(2) or any of the routines built on it (such as execv(3)), it will stop before executing the first instruction of the new image.  Also, any setuid or setgid bits on the executable being executed will be ignored.

The part I have emphasized is quite important: if the process ends up stopping for any reason, it will be impossible to start it again. When a process is being `ptrace`d, it will stop upon delivery of any signal (normally, so the parent process can respond appropriately) but in this cause launchd has no idea that we are being traced so it will not know how to handle it correctly. If our program crashes or is killed by the system, the process will not exit, and this will cause the *entire system* to slowly grind to a halt as (I think) it first tries to repeatedly `SIGKILL` your process, fails to do, and then just hangs in something important while waiting for process termination that will never come. One way to avoid this is to convert signals to Mach exceptions using the `PT_SIGEXC` `ptrace` request, and install a Mach exception handler to handle these:

```objc
#import <mach/mach.h>
#import <pthread.h>
#import <sys/sysctl.h>

#import "AppDelegate.h"

boolean_t exc_server(mach_msg_header_t *, mach_msg_header_t *);
int ptrace(int, pid_t, caddr_t, int);

#define PT_TRACE_ME 0
#define PT_SIGEXC 12

kern_return_t catch_exception_raise(mach_port_t exception_port,
                                    mach_port_t thread,
                                    mach_port_t task,
                                    exception_type_t exception,
                                    exception_data_t code,
                                    mach_msg_type_number_t code_count) {
	// Forward the request to the next-level Mach exception handler. This will
	// probably be ReportCrash's.
	return KERN_FAILURE;
}

void *exception_handler(void *argument) {
	mach_port_t port = *(mach_port_t *)argument;
	mach_msg_server(exc_server, 2048, port, 0);
	return NULL;
}

int main(void) {
	ptrace(PT_TRACE_ME, 0, NULL, 0);

	ptrace(PT_SIGEXC, 0, NULL, 0);

	mach_port_t port = MACH_PORT_NULL;
	mach_port_allocate(mach_task_self(), MACH_PORT_RIGHT_RECEIVE, &port);
	mach_port_insert_right(mach_task_self(), port, port, MACH_MSG_TYPE_MAKE_SEND);
	// PT_SIGEXC maps signals to EXC_SOFTWARE; note that this will interfere
	// with the debugger (which will try to do the same thing via PT_ATTACHEXC).
	// Usually you'd check for that and predicate the execution of the following
	// code on whether it's attached.
	task_set_exception_ports(mach_task_self(), EXC_MASK_SOFTWARE, port, EXCEPTION_DEFAULT, THREAD_STATE_NONE);
	pthread_t thread;
	pthread_create(&thread, NULL, exception_handler, (void *)&port);

	@autoreleasepool {
		return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
	}
}
```

While this won't catch `SIGKILL`, we can try to avoid being sent these by exiting before we'd get one in the cases were we can:

```objc
#import "AppDelegate.h"

@implementation AppDelegate
- (void)applicationWillTerminate:(UIApplication *)application {
	exit(0);
}
@end
```

Finally, this procedure is unfit for the App Store: not only does it use private API, it requires the process to have the `get-task-allow` entitlement, which Apple only grants for code signed with a development certificate. Apps of this type cannot be submitted to the App Store or TestFlight.
