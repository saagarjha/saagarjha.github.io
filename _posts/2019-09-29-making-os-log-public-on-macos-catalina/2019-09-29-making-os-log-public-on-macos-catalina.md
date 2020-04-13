---
layout: post
title: "Making `os_log` Public on macOS Catalina"
clean_title: "Making os_log Public on macOS Catalina"
---

Apple's [unified logging system](https://developer.apple.com/documentation/os/logging) launched in macOS Sierra and iOS 10 as the successor to [Apple System Logger](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/asl.3.html)–itself a replacement for the [syslog(3)](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/syslog.3.html) API–and alongside it came changes in how logs were organized and persisted, plus a new `log` command and redesigned Console app to make sense of these messages. One important change, however, [related to privacy](https://developer.apple.com/documentation/os/logging#1841411): by default, dynamic data not specifically annotated with the `%{public}` keyword will not be logged–the format specifier will be replaced with "&lt;private&gt;" instead of the identified object. While this helps prevent against accidentally leaking personal information, it can be inconvenient when debugging issues. To bypass this restriction, the `log` command used to allow for changing the logging configuration to display all data:

```console
$ sudo log config --status
System mode = DEFAULT
$ sudo log config --mode private_data:on
$ sudo log config --status
System mode = DEFAULT PRIVATE_DATA
```

Unfortunately, as of macOS Catalina this command no longer works:

```console
$ sudo log config --mode private_data:on
log: Invalid Modes 'private_data:on'
```

While `log` no longer lets us set this secret configuration option, it's not gone entirely: we can still enable it ourselves, just not directly. If you're impatient, you can skip directly to [here](#putting-it-all-together); otherwise read on to see how this setting works internally.

{% capture update_2_1_20_code %}
```console
$ codesign -d --entitlements :- `which log`
Executable=/usr/bin/log
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.private.debug_port</key>
	<true/>
	<key>com.apple.private.logging.admin</key>
	<true/>
	<key>com.apple.private.set-atm-diagnostic-flag</key>
	<true/>
</dict>
</plist>
```
{% endcapture %}
{% assign update_2_1_20="Unfortunately it appears that somewhere around macOS 10.15.3 Apple added an extra check for `host_set_atm_diagnostic_flag`, restricting its functionality to binaries with the `com.apple.private.set-atm-diagnostic-flag` entitlement: ![Hopper disassembly of the kernel code invoked by host_set_atm_diagnostic_flag, showing a new entitlements check](HopperEntitlementCheck.png) `log` of course possesses this new entitlement, " | append: update_2_1_20_code | append: "but since we cannot, any of our attempts to set the diagnostic flag will fail with `KERN_NO_ACCESS` (supposedly a [Bogus access restriction](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/osfmk/mach/kern_return.h#L109)…). If you are comfortable with disabling System Integrity Protection, the steps mentioned [here](https://saagarjha.com/blog/2019/09/29/making-os-log-public-on-macos-catalina#enabling-private_data-by-hand) will still work, but otherwise I don't see any easy ways to get around this new requirement." %}

{% include aside.html type="Update" date="2/1/20" content=update_2_1_20 %}

## You must be this tall to ride

As with any reverse-engineering exercise, we pull out our trusty disassembler and point it at `/usr/bin/log`. We're looking for the code that handles configurations, so it makes sense to look for the error we got earlier; searching for "Invalid Modes" gives us a string that we can cross-reference to a function that clearly handles parsing options for the `config` subcommand:

![Hopper disassembly of the configuration parsing function inside of log](HopperConfigParser.png)

It might not be completely clear what's going on here, but the gist is that it's going through the list of possible settings trying to match it against the one that comes from argument we pass in. Specifically, `var_30` contains the "private_data" part of "private_data:on", and `var_38` contains the second half. Assuming the following structure layouts,

```c
typedef struct {
	char *name; // e.g. "on"
	char not_important[16];
} mode_value;

typedef struct {
	char *name; // e.g. "private_data"
	mode_value *allowed_values;
} mode;
```

at `0x1000321c0` (which `var_50` contains) is a `mode[7]` embedded in the binary to test against: in order, their names are "level", "stream", "persist", "private_data", "sensitive_data", "legacy_logging", and "customer". We can make out that `r14` is a index into the array for the mode that's being checked. Now that we have this background, it's clear why we can't set "private_data": we can't go past the second option if `_os_trace_is_development_build` returns false, which it likely will considering we don't have access to a development build (a peek in the debugger confirms this suspicion). Previous versions of `log` would let us go up to four, providing access to the secret "persist" and "private_data" options, but with this change Apple has rendered them inaccessible.

To make sure that the private_data functionality is still present but gated behind the check, we can use the debugger to pretend that we are in fact using a development build and see if we can activate the option:

```console
$ sudo lldb `which log` -- config --mode private_data:on
(lldb) target create "/usr/bin/log"
Current executable set to '/usr/bin/log' (x86_64).
(lldb) settings set -- target.run-args  "--" "config" "--mode" "private_data:on"
(lldb) breakpoint set --name _os_trace_is_development_build
Breakpoint 1: where = libsystem_trace.dylib`_os_trace_is_development_build, address = 0x000000000000df57
(lldb) breakpoint command add
Enter your debugger command(s).  Type 'DONE' to end.
> thread return 1
> continue
(lldb) run
Process 27107 launched: '/usr/bin/log' (x86_64)
(lldb)  thread return 1
(lldb)  continue
Process 27107 resuming

Command #2 'continue' continued the target.
(lldb)  thread return 1
(lldb)  continue
Process 27107 resuming

Command #2 'continue' continued the target.
(lldb)  thread return 1
(lldb)  continue
Process 27107 resuming

Command #2 'continue' continued the target.
2019-09-29 04:20:17.637538-0700 log[27107:5793152] Changed system mode to 0x1000000
Process 27107 exited with status = 0 (0x00000000)
$ sudo log config --status
System mode = DEFAULT PRIVATE_DATA
```

## Enabling private_data by hand

Now that we know it's possible to set private_data from the `log` binary, we have a number of options. The first would be to write a dynamic library that interposes `_os_trace_is_development_build`, like so:

```c
int _os_trace_is_development_build();

int overriden__os_trace_is_development_build() {
	return 1; // Look at me, I'm the development build now
}

__attribute__((used, section("__DATA,__interpose"))) static struct {
	int (*overridden__os_trace_is_development_build)();
	int (*_os_trace_is_development_build)();
} _os_trace_is_development_build_interpose = {overriden__os_trace_is_development_build, _os_trace_is_development_build};
```

However, actually using it is somewhat of a pain, since setuid programs such as `sudo` strip out `DYLD_*` environment variables:

```console
$ clang force_development_build.c -shared -o force_development_build.dylib
$ sudo bash # So we don't need sudo
# DYLD_INSERT_LIBRARIES="$(pwd)/force_development_build.dylib" log config --mode private_data:on
# 
exit
$ sudo log config --status
System mode = DEFAULT PRIVATE_DATA
```

A more convenient technique would be to see what `log` does when it needs to change the configuration and write something that replicates this. Looking through the configuration parsing function, we stumble upon a call to a routine that looks promising:

![Hopper disassembly of the configuration setter function inside of log](HopperConfigurationSetter.png)

There's a peculiarity in this function that we'll get to in a bit; first, let's focus on `host_set_atm_diagnostic_flag`. [The documentation](https://developer.apple.com/documentation/kernel/1502446-host_set_atm_diagnostic_flag) for this kernel function is a bit laconic, but it's enough to tell us that it takes a `host_priv_t` and some sort of diagnostic flag that encompasses the logging configuration. Static or dynamic analysis shows that the system's private_data setting is controlled by bit 24 of `diagnostic_flag`. This lets us enable private_data, but we still have a small problem: we if we just send over `1 << 24` we're going to clobber all the other settings we have already configured. We need to find a way to modify the current configuration instead of overwriting it, and to do that we need to be able to get the current value of `diagnostic_flag`. A quick search turns up [`host_get_atm_diagnostic_flag`](https://developer.apple.com/documentation/kernel/1502446-host_set_atm_diagnostic_flag), so we have what we need to write our own implementation that sets private_data correctly. However, we have time for another brief tangent to wrap up some loose ends.

## The case of the missing `host_get_atm_diagnostic_flag`

Looking through the configuration-related code in `log`, you'll see something strange: there are no calls to `host_get_atm_diagnostic_flag`! In its place there are a number of loads from the seemingly-random address `0x7fffffe00048`: there's even one I alluded to in the function we examined earlier. *Something* must be mapped there, because otherwise the program would fault, but this address is way outside anything from the binary.  Hmm…`0x7fffffe00048`…`0x7fffffe00048`…the address is in the commpage!

The commpage on macOS serves a purpose similar to [vsyscall on Linux](https://lwn.net/Articles/446528/): that is, it's a chunk of data and code that's shared and mapped into every process at a fixed address and serves to reduce the number of roundtrips to the kernel. For 64-bit versions of macOS, the commpage base is located at `0x7fffffe00000`, which means that the address we're seeing is located inside of it. For performance reasons, it seems like Apple decided to proxy `diagnostic_flag` into userspace using this technique, where it joins CPU and high-resolution time information (this probably keeps the overhead for a call to `os_log` down). It's the kernel's job to maintain this information, and we can see how it does so by glancing at [the implementation of `host_set_atm_diagnostic_flag`](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/osfmk/kern/host.c#L1334): it calls [`atm_set_diagnostic_config`](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/osfmk/atm/atm.c#L1363), which in turn passes the configuration value to [`commpage_update_atm_diagnostic_config`](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/osfmk/i386/commpage/commpage.c#L813). This last function updates the four-byte `diagnostic_flag` at `_COMM_PAGE_ATM_DIAGNOSTIC_CONFIG`, which [we can see](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/osfmk/i386/cpu_capabilities.h#L200) is `(_COMM_PAGE_START_ADDRESS+0x48)` as we'd expect. ([`host_get_atm_diagnostic_flag`'s implementation](https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/libsyscall/mach/host.c#L37) is nothing special, either–it just reads from this address.)

## Putting it all together

With all that done, we can write a simple tool to update private_data. Make sure to run it as root, or it will exit with `KERN_INVALID_ARGUMENT` (== 4).

```c
#include <mach/mach_host.h>
#include <stdint.h>
#include <stdio.h>

const uint32_t private_data_flag = 1 << 24;

int main(int argc, char **argv) {
	if (argc == 2 && ++argv) {
		uint32_t diagnostic_flag;
		host_get_atm_diagnostic_flag(mach_host_self(), &diagnostic_flag);
		if (!strcmp(*argv, "status")) {
			puts(diagnostic_flag & private_data_flag ? "enabled" : "disabled");
			return 0;
		} else if (!strcmp(*argv, "enable")) {
			return host_set_atm_diagnostic_flag(mach_host_self(), diagnostic_flag | private_data_flag);
		} else if (!strcmp(*argv, "disable")) {
			return host_set_atm_diagnostic_flag(mach_host_self(), diagnostic_flag & ~private_data_flag);
		}
	} else {
		fprintf(stderr, "Usage: %s <status|enable|disable>\n", *argv);
	}
}
```

{% include aside.html type="Warning" content="I'm not sure if I'm supposed to call `mach_port_deallocate` on `mach_host_self`, but `log` doesn't seem to so I'm going to let it be for now. If you start seeing crashes with `__THE_SYSTEM_HAS_NO_PORT_SETS_AVAILABLE__` in the backtrace, let me know and I'll update this code." %}
