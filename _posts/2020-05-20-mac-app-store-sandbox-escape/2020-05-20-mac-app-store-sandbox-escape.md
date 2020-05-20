---
layout: post
title: "Mac App Store Sandbox Escape"
---

The [App Sandbox](https://developer.apple.com/library/archive/documentation/Security/Conceptual/AppSandboxDesignGuide/AboutAppSandbox/AboutAppSandbox.html), originally introduced in Mac OS X Leopard as "the Seatbelt", is a macOS security feature modeled after [FreeBSD's Mandatory Access Control](https://www.freebsd.org/doc/handbook/mac.html) (left unabbreviated for clarity) that serves as a way to restrict the abilities of an application beyond the usual user- and permission-based systems that UNIX offers. The full extent of the capabilities the sandbox manages is fairly broad, ranging from file operations to Mach calls, and is specified in a custom Scheme implementation called the Sandbox Profile Language (SBPL). The sandbox profiles that macOS ships with can be found in /System/Library/Sandbox/Profiles, and while their format is technically SPI (as the header comment on them will tell you) there is fairly extensive [third-party documentation](https://reverse.put.as/wp-content/uploads/2011/09/Apple-Sandbox-Guide-v1.0.pdf). The implementation details of sandboxing are not intended to be accessed by third-party developers, but applications on Apple's platforms can request (and in some cases, such as new applications distributed on the Mac App Store and all applications for Apple's embedded platforms, must function in) a sandbox specified by a fixed, system-defined profile (on macOS, application.sb). Barring [a few exceptions](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/AppSandboxTemporaryExceptionEntitlements.html) (which usually require additional review and justification for their use) this system-provided sandbox provide an effective way to prevent applications from accessing user data without consent or performing undesired system modifications.

In January I discovered a flaw in the implementation of the sandbox initialization procedure on macOS that would allow malicious applications distributed through the Mac App Store to circumvent the enforcement of these restrictions and silently perform unauthorized operations, including actions such as accessing sensitive user data. Apple has since implemented changes in the Mac App Store to address this issue and the technique outlined below should no longer be effective.

## Sandbox initialization on macOS

Sandboxing is enforced by the kernel and present on both macOS and Apple's iOS-based operating systems, but it is important to note that third party code is not required to run in a sandbox on macOS. While the use of the platform sandbox is mandatory for third-party software running on embedded devices, on Macs it is rarely used by applications distributed outside of the Mac App Store; even on the store there are still a couple of unsandboxed applications that have been grandfathered into being allowed to remain for sale as they were published prior to the 2012 sandboxing deadline. A lesser known, but likely related fact is that processes are not born sandboxed on macOS: unlike iOS, where the sandbox is applied by the kernel before the first instruction of a program executes, on macOS a process must elect to place itself into the sandbox using the "deprecated" `sandbox_init(3)` family of functions. These themselves are wrappers around the `__sandbox_ms` function, an alias for `__mac_syscall` from libsystem_kernel.dylib in /usr/lib/system. This design raises an important question: if a process *chooses* to place itself in a sandbox, how does Apple *require* it for apps distributed through the Mac App Store?

Experienced Mac developers already know the answer: Apple checks for the presence of the `com.apple.security.app-sandbox` entitlement in all apps submitted for review, and its mere existence magically places the process in a sandbox by the time code execution reaches `main`. But the process isn't actually magic at all: it's performed by a function called `_libsecinit_initializer` inside the library libsystem_secinit.dylib, also located at /usr/lib/system:

![libsystem_secinit.dylib opened in Hopper, showing _libsecinit_initializer](Hopper_libsecinit_initializer.png)

`_libsecinit_initializer` calls `_libsecinit_appsandbox`, which (among other things) copies the current process's entitlements, checks for the `com.apple.security.app-sandbox` in them, and calls `__sandbox_ms` after consulting with the secinitd daemon. So this answers *where* the sandbox is applied, but doesn't explain *how*: for that, we need to look inside libSystem.

libSystem is the standard C library on macOS (see `intro(3)` for more details). While it vends system APIs, by itself it does very little; instead, it provides this functionality by re-exporting all the libraries inside of /usr/lib/system:

```console
$ otool -L /usr/lib/libSystem.dylib
/usr/lib/libSystem.dylib:
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
	/usr/lib/system/libcache.dylib (compatibility version 1.0.0, current version 83.0.0)
	/usr/lib/system/libcommonCrypto.dylib (compatibility version 1.0.0, current version 60165.120.1)
	/usr/lib/system/libcompiler_rt.dylib (compatibility version 1.0.0, current version 101.2.0)
	/usr/lib/system/libcopyfile.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/system/libcorecrypto.dylib (compatibility version 1.0.0, current version 866.120.3)
	/usr/lib/system/libdispatch.dylib (compatibility version 1.0.0, current version 1173.100.2)
	/usr/lib/system/libdyld.dylib (compatibility version 1.0.0, current version 750.5.0)
	/usr/lib/system/libkeymgr.dylib (compatibility version 1.0.0, current version 30.0.0)
	/usr/lib/system/liblaunch.dylib (compatibility version 1.0.0, current version 1738.120.8)
	/usr/lib/system/libmacho.dylib (compatibility version 1.0.0, current version 959.0.1)
	/usr/lib/system/libquarantine.dylib (compatibility version 1.0.0, current version 110.40.3)
	/usr/lib/system/libremovefile.dylib (compatibility version 1.0.0, current version 48.0.0)
	/usr/lib/system/libsystem_asl.dylib (compatibility version 1.0.0, current version 377.60.2)
	/usr/lib/system/libsystem_blocks.dylib (compatibility version 1.0.0, current version 74.0.0)
	/usr/lib/system/libsystem_c.dylib (compatibility version 1.0.0, current version 1353.100.2)
	/usr/lib/system/libsystem_configuration.dylib (compatibility version 1.0.0, current version 1061.120.2)
	/usr/lib/system/libsystem_coreservices.dylib (compatibility version 1.0.0, current version 114.0.0)
	/usr/lib/system/libsystem_darwin.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/system/libsystem_dnssd.dylib (compatibility version 1.0.0, current version 1096.100.3)
	/usr/lib/system/libsystem_featureflags.dylib (compatibility version 1.0.0, current version 17.0.0)
	/usr/lib/system/libsystem_info.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/system/libsystem_m.dylib (compatibility version 1.0.0, current version 3178.0.0)
	/usr/lib/system/libsystem_malloc.dylib (compatibility version 1.0.0, current version 283.100.6)
	/usr/lib/system/libsystem_networkextension.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/system/libsystem_notify.dylib (compatibility version 1.0.0, current version 241.100.2)
	/usr/lib/system/libsystem_sandbox.dylib (compatibility version 1.0.0, current version 1217.120.7)
	/usr/lib/system/libsystem_secinit.dylib (compatibility version 1.0.0, current version 62.100.2)
	/usr/lib/system/libsystem_kernel.dylib (compatibility version 1.0.0, current version 6153.121.1)
	/usr/lib/system/libsystem_platform.dylib (compatibility version 1.0.0, current version 220.100.1)
	/usr/lib/system/libsystem_pthread.dylib (compatibility version 1.0.0, current version 416.100.3)
	/usr/lib/system/libsystem_symptoms.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/system/libsystem_trace.dylib (compatibility version 1.0.0, current version 1147.120.0)
	/usr/lib/system/libunwind.dylib (compatibility version 1.0.0, current version 35.4.0)
	/usr/lib/system/libxpc.dylib (compatibility version 1.0.0, current version 1738.120.8)
```

Like most standard libraries, the compiler will automatically (and dynamically) link it into your programs even if you don't specify it explicitly:

```console
$ echo "int main(void) {}" | clang -x c - && otool -L a.out
a.out:
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
```

When a program is started, the dynamic linker will [ensure that libSystem's initializer functions](https://github.com/apple-open-source-mirror/dyld/blob/f033f5564c85c5cbfd24cf25e702e4bb0c2c39b4/dyld3/AllImages.cpp#L1627) are called, which includes [the function that calls `_libsecinit_initializer`](https://github.com/apple-open-source-mirror/Libsystem/blob/691891bd298f36bf91780ca4f3a2a2cd57b17999/init.c#L259). As dyld ensures that libSystem's initializer is run prior to handing off control to the app's code, this ensures that any application that links against it will have sandboxing applied to it before it can execute its own code.

## Bypassing sandbox initialization

As you may have guessed, this process is problematic. In fact, there are actually multiple issues, each of which allows an application with the `com.apple.security.app-sandbox` entitlement to bypass the sandbox initialization process.

### dyld interposing

dyld interposing is a neat little feature that allows applications to tell the dynamic linker to "interpose" an exported function and replace it with another by including a special `__DATA,__interpose` section in their binary. Since `_libsecinit_appsandbox` is exported by libsystem_secinit.dylib so that it can be called by libSystem, we can try interposing it with a function that does nothing:

```c
void _libsecinit_initializer(void);

void overriden__libsecinit_initializer(void) {
}

__attribute__((used, section("__DATA,__interpose"))) static struct {
	void (*overriden__libsecinit_initializer)(void);
	void (*_libsecinit_initializer)(void);
} _libsecinit_initializer_interpose = {overriden__libsecinit_initializer, _libsecinit_initializer};
```

When interposing was first introduced, it would only be applied when a library was preloaded into a process using the `DYLD_INSERT_LIBRARIES` environment variable. However, on newer OSes this functionality has been improved to [work for any linked libraries as well](https://github.com/apple-open-source-mirror/dyld/blob/f033f5564c85c5cbfd24cf25e702e4bb0c2c39b4/src/dyld2.cpp#L6605), which means all we have to do to take advantage of this feature is put this code in a framework and link against it in our main app. Since [interposing is applied before image initializers](https://github.com/apple-open-source-mirror/dyld/blob/f033f5564c85c5cbfd24cf25e702e4bb0c2c39b4/src/dyld2.cpp#L6679) we will be able to prevent the real `_libsecinit_initializer` from running and thus `__sandbox_ms` being called. Success!

As this technique allowed an application that appears to be sandboxed (possessing the `com.apple.security.app-sandbox` entitlement) to interfere with its own initialization process, I reported this issue to Apple on January 20th and explained that such an app might be able to be submitted to the App Store and get past app review. On March 19th, I received a reply from Apple stating that App Store applications are prevented from being interposed, which was news to me. Apparently right after I submitted my original report Apple added an additional check in dyld, one so new that it's still not in any public sources:

![Hopper disassembly of dyld::_main, focused on code inlined from configureProcessRestrictions, highlighting the existence of a new AMFI flag](HopperAdditionaldyldRestrictions.png)

While the [dyld source for `configureProcessRestrictions`](https://github.com/apple-open-source-mirror/dyld/blob/f033f5564c85c5cbfd24cf25e702e4bb0c2c39b4/src/dyld2.cpp#L5060) only shows five flags being read from `amfi_check_dyld_policy_self`, the binary clearly checks a sixth: `1 << 6`. (`configureProcessRestrictions` has been inlined here into its caller, `dyld::_main`.) I still do not know what its real name is but it's used later in `dyld::_main` to control whether interposing is allowed. This means we can't interpose `_libsecinit_initializer`–we'll have to prevent it from from being called instead.

### Static linking

Linking against libSystem causes dyld to call `_libsecinit_initializer`, so it's logical to try to avoid having anything to do with dyld at all. This is fairly strange to do on macOS, as it does not have a stable syscall interface, but with the right set of compiler flags we can make a fully static binary that needs to no additional support to run.

{% include aside.html type="Fun fact" content="We know that this must be possible because Go used to do it, until they got tired of new versions of macOS breaking everything and gave up trying to make system calls themselves. Go programs [as of Go 1.11](https://golang.org/doc/go1.11#runtime) now use libSystem." %}

Unfortunately, macOS does not ship with a crt0.o that we can statically link, so using just the `-static` flag does not work:

```console
$ echo "int main() {}" | clang -x c -static -
ld: library not found for -lcrt0.o
clang: error: linker command failed with exit code 1 (use -v to see invocation
```

But if we're jettisoning the standard library, we might as well get rid of the C runtime as well, defining our own `start` symbol:

```console
$ clang -x c -static -nostdlib -
void start(void) __asm__("start");

void start(void) {
        while (1);
}
$ otool -L a.out
a.out:
$ a.out
^C
$
```

No dyld means no code that can arrange a call `_libsecinit_initializer`, so we're free to do whatever we like without restriction. However, not having libSystem and dyld to support us means we cannot use dynamic linking and need to make raw system calls for everything, which is a bit of a pain. One way to resolve this would be to keep the unsandboxed code short–just a couple of calls to acquire a handle on restricted resources–then stash that away before `execve`ing a new dynamically linked binary, restoring the process to a sane state. When responding to Apple with a new sample program based on this idea, I simply `open`ed a file descriptor for the home directory (you can locate the directory without any syscalls by pulling the current username from the `apple` array on the stack during process initialization) and then once that succeeds executed an inner binary. The new file descriptor was preserved for across the `execve` call and became accessible to the inner application, even though that one was dynamically linked and had the sandbox applied to it as usual.

### Dynamically linking against nothing

Statically linking works, but it's somewhat inconvenient: either you perform the work of the dynamic linker yourself if you want to do anything non-trivial, or you `execve` a new binary. It's actually worse than that though, because there's an additional complication: executing a new binary causes a hook in the AppleMobileFileIntegrity kernel extension to run, and when System Integrity Protection is enabled this hook (for reasons unknown to me) checks to see if the process has a valid dyld signature:

![Hopper disassembly of the MAC hook _cred_label_update_execve, showing the check for CS_DYLD_PLATFORM](HopperAMFIHook.png)

The strange pointer arithmetic and mask is really a check for [`CS_DYLD_PLATFORM`](https://github.com/apple-open-source-mirror/xnu/blob/65d5be84e21bccbae44730ddb3b307a701b0351a/osfmk/kern/cs_blobs.h#L62), which the comment helpfully states is set if the "dyld used to load this is a platform binary". Since we didn't use dyld at all, this isn't set and we can't `execve`. While  malicious applications willing to do a bit of work can still "fix" their process without blowing it away, I figured I might as well figure out a way to construct a new one.

Since the hook wants us to have a valid dyld, we should probably just link dynamically. As we mentioned before, this makes the compiler automatically bring in libSystem (and with it, the libsystem_secinit.dylib initializers), which we don't want. I couldn't find out a way to get the linker to not automatically insert the load command for libSystem, but we can get essentially the same result by modifying the binary ourselves afterwards to delete that specific command. [I found a Mach-O editor online](https://github.com/Tyilo/macho_edit) that was slightly crashy but worked well enough for this purpose. Unfortunately, removing the load command isn't enough: dyld [specifically checks](https://github.com/apple-open-source-mirror/dyld/blob/f033f5564c85c5cbfd24cf25e702e4bb0c2c39b4/src/dyld2.cpp#L6703) for libSystem "glue" before running our code, and as we don't have a libSystem at all it aborts execution.

However, there's one way around this: if we use a `LC_UNIXTHREAD` rather than a `LC_MAIN` load command, dyld will pass execution to us without checking for libSystem (as it thinks we have linked against crt1.o instead). Both load commands specify the entrypoint of the executable, but `LC_MAIN` is the "new" way of doing so. `LC_UNIXTHREAD` specifies the entire thread state, but `LC_MAIN` only points to the "entry offset" where code execution should begin–the linker sets this to where `main` is, unless you've used `-e` to change it. The compiler uses it for dynamically linked binaries because it expects libSystem to set all the thread state prior to calling the entrypoint function.

```console
$ echo "int main(void) {}" | clang -x c -
$ nm a.out
0000000100000000 T __mh_execute_header
0000000100000fb0 T _main
                 U dyld_stub_binder
$ otool -l a.out | grep -A 3 "LC_MAIN"
       cmd LC_MAIN
   cmdsize 24
  entryoff 4016
 stacksize 0
$ clang -x c -static -nostdlib -
void start(void) __asm__("start");

void start(void) {
        while (1);
}
$ nm a.out
0000000100000000 A __mh_execute_header
0000000100000fb0 T start
$ otool -l a.out | grep -A 11 "LC_UNIXTHREAD"
        cmd LC_UNIXTHREAD
    cmdsize 184
     flavor x86_THREAD_STATE64
      count x86_THREAD_STATE64_COUNT
   rax  0x0000000000000000 rbx 0x0000000000000000 rcx  0x0000000000000000
   rdx  0x0000000000000000 rdi 0x0000000000000000 rsi  0x0000000000000000
   rbp  0x0000000000000000 rsp 0x0000000000000000 r8   0x0000000000000000
    r9  0x0000000000000000 r10 0x0000000000000000 r11  0x0000000000000000
   r12  0x0000000000000000 r13 0x0000000000000000 r14  0x0000000000000000
   r15  0x0000000000000000 rip 0x0000000100000fb0
rflags  0x0000000000000000 cs  0x0000000000000000 fs   0x0000000000000000
    gs  0x0000000000000000
```

The linker flag `-no_new_main` tells the linker to use `LC_UNIXTHREAD` instead of `LC_MAIN` for dynamically linked executables, but it has been silently ignored for years (apparently, this has something to do with [rdar://problem/39514191](rdar://problem/39514191)). This means to generate the binary we'll have to go back in time and download an old toolchain that accepts this flag. The one that [Xcode 5.1.1](https://download.developer.apple.com/Developer_Tools/xcode_5.1.1/Xcode_5.1.1.dmg) ships with does nicely.

{% include aside.html type="Fun fact" content="I have a friend who refers to Xcode 5.1.1 as \"the good toolchain\" because it  supports everything." %}

Once we use that to create a binary, upon running it we have a valid dyld in our process and unsandboxed code execution so we can just continue as we did in the statically linked case, as this will satisfy AMFI's checks.

## Final thoughts

I submitted the final example to Apple just before the initial 90-day disclosure deadline of April 20th, and when they requested an extension to work on the new information I provided them with an additional 30 days. Apple says it has made changes in the Mac App Store to address this issue during that period, and although I [don't really have a good way to check](https://twitter.com/0xcharlie/status/133680514950369280) if or how the change works I would guess that it simply looks for and rejects applications using techniques similar to the ones described above.

dyld is a fairly complicated system and it has many useful features, but these features along with the fact that it runs in-process makes it nontrivial to protect against control flow subversion early in the initialization process. Applying sandboxing in the kernel itself, as iOS does, is probably a better solution in the long run, as the bugs I found here were fairly straightforwards and exploited logic errors rather than undefined behavior in the language. Perhaps we will see such a change in the future.

The code I submitted to Apple to demonstrate the issue is [available online](https://github.com/saagarjha/macOSSandboxInitializationBypass).

## Timeline

* **1/20/20:** Initial disclosure of library interposing bypass to Apple
* **1/22/20:** Acknowledgment of submission by Apple
* **1/28/20:** Request for status update after recent updates did not resolve the issue
* **1/29/20:** Response from Apple that they were still investigating
* **2/26/20:** Request for update and affirmation of 90-day disclosure timeline
* **2/28:20:** Response from Apple that they were still looking into the issue
* **3/19:20:** Email from Apple stating that Mac App Store applications cannot be interposed
* **3/20/20:** Submission of statically linked application to avoid interposing
* **3/23/23:** Acknowledgement of the new information
* **4/14/20:** Submission of dynamically linked application to bypass `execve` limitation
* **4/17/20:** Request for more time from Apple to analyze the new submission
* **4/19/20:** Disclosure deadline extended by 30 days to May 20th
* **4/20/20:** Confirmation and appreciation for the extension
* **5/13/20:** Request for an update on progress
* **5/15/20:** Confirmation that a change had been implemented in Mac App Store
* **5/20/20:** Expiration of discretionary disclosure extension
