---
layout: post
title: "Bypassing `objc_msgSend`"
clean_title: "Bypassing objc_msgSend"
---

On its fastest path, [`objc_msgSend`](https://developer.apple.com/documentation/objectivec/1456712-objc_msgsend) can transfer execution to an [`IMP`](https://developer.apple.com/documentation/objectivec/objective-c_runtime/imp) in just over a dozen instructions:

```console
$ otool -tV /usr/lib/libobjc.dylib -p _objc_msgSend | head -n 18
/usr/lib/libobjc.dylib:
(__TEXT,__text) section
_objc_msgSend:
0000000000006e00	testq	%rdi, %rdi
0000000000006e03	je	0x6e78
0000000000006e06	testb	$0x1, %dil
0000000000006e0a	jne	0x6e83
0000000000006e0d	movabsq	$_objc_absolute_packed_isa_class_mask, %r10
0000000000006e17	andq	__objc_empty_vtable(%rdi), %r10
0000000000006e1a	movq	%rsi, %r11
0000000000006e1d	andl	0x18(%r10), %r11d
0000000000006e21	shlq	$0x4, %r11
0000000000006e25	addq	0x10(%r10), %r11
0000000000006e29	cmpq	__objc_empty_vtable(%r11), %rsi
0000000000006e2c	jne	0x6e38
0000000000006e2e	movq	0x8(%r11), %r11
0000000000006e32	xorq	%r10, %r11
0000000000006e35	jmpq	*%r11
```

{% include aside.html content="You can ignore `__objc_empty_vtable`; `otool` is confused because the loads have no displacement and so it chooses to decorate them with [the symbol at absolute address zero](https://github.com/apple-open-source-mirror/objc4/blob/2bd4d4a5c3e0f853a5d4fd8ae762619abf759cdf/runtime/objc-cache.mm#L148)." %}

As one of the hottest, if not *the* hottest code paths on the system, it has good reason for being so optimized: even the most minimal savings here translate directly to measurable performance increases across all Objective-C applications. But even as fast as this implementation is, sometimes it's just not good enough: Objective-C is a dynamic language, and its features come with a price even if you're not using them. The majority of the code executed in a typical `objc_msgSend` call is not for implementation lookup: rather, it's checks to see if execution can proceed on the fast path. But the checks are comparatively cheap: an even bigger issue is that the function largely serves as an optimization barrier to the compiler, to which it looks like nothing more than an impenetrable trampoline to an unknown target. As such, even the simplest member access cannot be inlined; every method call *must* clobber multiple registers and almost always requires a move out of the accumulator, where it is usually undesirable to leave the result.

[`__attribute__((objc_direct))`](https://github.com/llvm/llvm-project/commit/d4e1ba3fa9dfec2613bdcc7db0b58dea490c56b1) (and the closely related `__attribute__((objc_direct_members))`) attempts to solve some of these problems by converting the method call into what's essentially C-style static dispatch. The change was met with some controversy when it was introduced: calls would no longer go through the runtime machinery, breaking many features of Objective-C, including swizzling, KVO, and even subclassing. Luckily, there's another way to get performance benefits without losing some of Objective-C's best features.

## A little bit of idle speculation

The key insight is that we can guess where the vast majority of method calls will go, and we can do so *at compile time* just by looking at the type of the receiver and message we're sending to it. Most of the time, `objc_msgSend` is just confirming what we already know: the only practical difference between its fast path and a direct function call is that it needs to check, as quickly as possible, that its target is what it expects it to be before jumping there. There's no way around this if we want to support Objective-C's dynamism; any implementation we come up with will have to do this too. The advantage we have over `objc_msgSend` is that as humans (or as a compiler), we have static type information and an understanding of the surrounding code which lets us guess, statically and with high accuracy, what the target is. In fact, we can just *speculate* that the call will go to the predicted method, and, taking a leaf out of `objc_msgSend`'s book, wrap a direct call to it in the barest minimum of checking to make sure that we were correct in our prediction. In psuedocode, we can convert a call to `-[Foo bar]` (where the reciever is `foo`) to something like this:

```
if (target is -[Foo bar]) {
	jump to -[Foo bar]
} else {
	objc_msgSend(foo, @selector(bar))
}
```

If we're correct, not only does this skip the cache lookup that `objc_msgSend` performs, but it also makes a *direct* call to the method: one which the compiler knows about and can optimize around.

## Jumping to an implementation

We're almost done: the only question that's left is what goes in the "target is `-[Foo bar]`" condition. As far as I can tell, if we have a `Foo *foo` and send `@selector(bar)` to it, the only cases where execution will not directly proceed to `-[Foo bar]` are if

1. the reciever is `nil`,
2. the reciever is a subclass of `Foo`, or
3. `-[Foo bar]` has been swizzled.

Checking for the first can be done easily; the second is a bit more complicated. In theory we're asking `[foo isMemberOfClass:Foo.class]`, but for obvious reasons we don't really want the overhead that comes with this. We can skip even a call to [`object_getClass`](https://developer.apple.com/documentation/objectivec/1418629-object_getclass?language=objc) by directly ripping the `isa` out of `foo` and comparing it against runtime metadata. There are some subtleties with [non-pointer `isa`s](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html) and [tagged pointers](https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html) but we can avoid them entirely by masking the `isa` appropriately and sending nonstandard cases to the slow path if we encounter them.

The last case, where the method has been swizzled, can't be determined by the information available to `objc_msgSend` alone: instead, we need to keep around a side table of boolean values that we must check on each call. When a method is swizzled the runtime (or your code interposed in front of the runtime) must update the corresponding entry so subsequent calls will go through the runtime.

{% include aside.html type="Update" date="1/5/20" content="Jean-Daniel Dupas has pointed out to me via email that invalidation is not quite as simple as I originally made it out to be, since [`method_exchangeImplementations`](https://developer.apple.com/documentation/objectivec/1418769-method_exchangeimplementations) takes two [`Method`](https://developer.apple.com/documentation/objectivec/method) parameters, which [lack back pointers](https://github.com/apple-open-source-mirror/objc4/blob/2bd4d4a5c3e0f853a5d4fd8ae762619abf759cdf/runtime/objc-runtime-new.h#L245) to the class they come from. The Objective-C runtime deals with this by [invalidating method caches for *all* classes](https://github.com/apple-open-source-mirror/objc4/blob/2bd4d4a5c3e0f853a5d4fd8ae762619abf759cdf/runtime/objc-runtime-new.mm#L3349) and letting them refill as as [the uncached `objc_msgSend` path](https://github.com/apple-open-source-mirror/objc4/blob/2bd4d4a5c3e0f853a5d4fd8ae762619abf759cdf/runtime/Messengers.subproj/objc-msg-simulator-x86_64.s#L995) [looks them up again](https://github.com/apple-open-source-mirror/objc4/blob/2bd4d4a5c3e0f853a5d4fd8ae762619abf759cdf/runtime/Messengers.subproj/objc-msg-simulator-x86_64.s#L384) and [adds them back](https://github.com/apple-open-source-mirror/objc4/blob/2bd4d4a5c3e0f853a5d4fd8ae762619abf759cdf/runtime/objc-runtime-new.mm#L5378). A real implementation would need support from the runtime to do something similar: it'll mark all entries in the table as invalid upon swizzling, forcing all method calls to go through (the slow path of) `objc_msgSend`, and start unsetting the entries as the uncached path verifies that the target has not been swizzled so they can go back through the optimized path." %}

With all the checks in place, we can now call the method directly. To try this out, [I patched Clang](https://github.com/saagarjha/expresscall) so that it would emit LLVM IR that performed the operations described above whenever it would have emitted a simple message send. I've never touched the LLVM project before, so my implementation is clumsy and probably incorrect; however, it works fine for the simple programs I tested it with and you can try it out yourself if compiling LLVM is your thing. Since I needed a name for the repository that wasn't just "llvm-project", I am henceforth naming this technique "expresscall"–as in "making an expresscall". "swiftcall" would have been shorter, but with significantly more potential to be confusing.

{% include aside.html type="Aside" content="I had originally started by writing a transformation pass to convert method calls by detecting calls to `objc_msgSend`, but unfortunately it turned out that the code generator for message sends erases the type information necessary to predict the target. So I ended up having to patch the code generator itself to gain access to this information." %}

## Are we fast yet?

There's no better way to see how fast (or slow) an optimization is than a highly synthetic benchmark. This one creates a simple Objective-C array type for integers, fills it with random numbers, and sums them. Since I'm a horrible person, you get to scroll past it:

```objc
@import Foundation;
@import ObjectiveC;
#import <algorithm>
#import <array>
#import <chrono>
#import <dlfcn.h>
#import <iostream>
#import <random>
#import <type_traits>
#import <unordered_map>
#import <utility>

const auto benchmark_size = 1000000;
const auto trials = 100;

@interface IntegerArray : NSObject
- (instancetype)initWithNumbers:(NSUInteger *)numbers count:(NSUInteger)count;
- (NSUInteger)count;
- (NSUInteger)numberAtIndex:(NSUInteger)index;
- (NSUInteger)direct_count __attribute__((objc_direct));
- (NSUInteger)direct_numberAtIndex:(NSUInteger)index __attribute__((objc_direct));
- (NSUInteger)swizzled_count;
- (NSUInteger)swizzled_numberAtIndex:(NSUInteger)index;
@end

@implementation IntegerArray {
	NSUInteger *_numbers;
	NSUInteger _count;
}

- (instancetype)initWithNumbers:(NSUInteger *)numbers count:(NSUInteger)count {
	if (self = [super init]) {
		_numbers = numbers;
		_count = count;
	}
	return self;
}

- (NSUInteger)count {
	return _count;
}

- (NSUInteger)numberAtIndex:(NSUInteger)index {
	return _numbers[index];
}

- (NSUInteger)direct_count {
	return _count;
}

- (NSUInteger)direct_numberAtIndex:(NSUInteger)index {
	return _numbers[index];
}

- (NSUInteger)swizzled_count {
	return 0;
}

- (NSUInteger)swizzled_numberAtIndex:(NSUInteger)index {
	return index;
}
@end

enum Benchmark {
	expresscall = 0,
	message_send,
	direct,
	swizzled_expresscall,
	swizzled_message_send,
	_end
};

auto measure(Benchmark benchmark, std::mt19937 &generator) {
	std::array<NSUInteger, benchmark_size> numbers;
	std::uniform_int_distribution<NSUInteger> distribution(0, NSUIntegerMax / benchmark_size);
	std::generate(numbers.begin(), numbers.end(), [&]() {
		return distribution(generator);
	});
	IntegerArray *array = [[IntegerArray alloc] initWithNumbers:numbers.data() count:numbers.size()];
	NSUInteger sum = 0;
	auto time = std::chrono::system_clock::now();
	switch (benchmark) {
	case Benchmark::expresscall:
		for (NSUInteger i = 0; i < array.count; ++i) {
			sum += [array numberAtIndex:i];
		}
		break;
	case Benchmark::message_send:
		for (NSUInteger i = 0; i < reinterpret_cast<NSUInteger (*)(IntegerArray *, SEL)>(objc_msgSend)(array, @selector(count)); ++i) {
			sum += reinterpret_cast<NSUInteger (*)(IntegerArray *, SEL, NSUInteger)>(objc_msgSend)(array, @selector(numberAtIndex:), i);
		}
		break;
	case Benchmark::direct:
		for (NSUInteger i = 0; i < array.direct_count; ++i) {
			sum += [array direct_numberAtIndex:i];
		}
		break;
	case Benchmark::swizzled_expresscall:
		for (NSUInteger i = 0; i < array.swizzled_count; ++i) {
			sum += [array swizzled_numberAtIndex:i];
		}
		break;
	case Benchmark::swizzled_message_send:
		for (NSUInteger i = 0; i < reinterpret_cast<NSUInteger (*)(IntegerArray *, SEL)>(objc_msgSend)(array, @selector(swizzled_count)); ++i) {
			sum += reinterpret_cast<NSUInteger (*)(IntegerArray *, SEL, NSUInteger)>(objc_msgSend)(array, @selector(swizzled_numberAtIndex:), i);
		}
		break;
	default:
		assert(false);
	}
	// Make sure the program actually does the calculation by returning the sum
	return std::make_pair(std::chrono::duration_cast<std::chrono::microseconds>(std::chrono::system_clock::now() - time), sum);
}

int main(void) {
	// Swizzle the dummy methods to point at actual implementations
	auto IntegerArray_count = class_getInstanceMethod(IntegerArray.class, @selector(count));
	class_replaceMethod(IntegerArray.class, @selector(swizzled_count), method_getImplementation(IntegerArray_count), method_getTypeEncoding(IntegerArray_count));
	*static_cast<bool *>(dlsym(dlopen(NULL, RTLD_LAZY), "OBJC_EXPRESSCALL____IntegerArray_swizzled_count_")) = true;
	auto IntegerArray_numberAtIndex_ = class_getInstanceMethod(IntegerArray.class, @selector(numberAtIndex:));
	class_replaceMethod(IntegerArray.class, @selector(swizzled_numberAtIndex:), method_getImplementation(IntegerArray_numberAtIndex_), method_getTypeEncoding(IntegerArray_numberAtIndex_));
	*static_cast<bool *>(dlsym(dlopen(NULL, RTLD_LAZY), "OBJC_EXPRESSCALL____IntegerArray_swizzled_numberAtIndex__")) = true;

	std::array<Benchmark, Benchmark::_end * trials> benchmarks;
	std::unordered_map<Benchmark, std::chrono::microseconds> times;
	std::random_device random;
	std::mt19937 generator(random());

	for (auto i = 0; i != Benchmark::_end; ++i) {
		std::fill(benchmarks.begin() + i * trials, benchmarks.begin() + (i + 1) * trials, Benchmark(i));
		times[Benchmark(i)] = std::chrono::microseconds(0);
	}
	// Randomize the order we run the benchmark iterations
	std::shuffle(benchmarks.begin(), benchmarks.end(), generator);

	std::cout << "Benchmarking average time to sum " << benchmark_size << " numbers (over " << trials << " trials)…" << std::endl;

	for (auto benchmark : benchmarks) {
		times[benchmark] += measure(benchmark, generator).first;
	}

	auto names = {"expresscall", "objc_msgSend", "objc_direct", "swizzled expresscall", "swizzled objc_msgSend"};
	for (auto i = 0; i != Benchmark::_end; ++i) {
		std::cout << *(names.begin() + i) << ": " << times[Benchmark(i)].count() / trials << "µs" << std::endl;
	}
	return 0;
}
```

We measure the time it takes for it to do this for both our optimized implementation and a normal method call (simulated by calling `objc_msgSend` directly, to prevent the compiler from converting it to its optimized form). For good measure, we throw in an `__attribute__((objc_direct))` implementation as well as swizzled versions of the other two tests as well (where we simulate the runtime setting the flag for expresscall to indicate it should abandon trying to directly call the method). Finally, in a reasonable-sounding but probably misguided attempt at preventing the CPU's branch predictor or cache from doing funny things, we randomize the order of the tests. On my computer, a [MacBook Pro (Retina, 13-inch, Early 2015)](https://support.apple.com/kb/SP715?locale=en_US) with an [Intel(R) Core(TM) i5-5287U CPU @ 2.90GHz](https://ark.intel.com/content/www/us/en/ark/products/84988/intel-core-i5-5287u-processor-3m-cache-up-to-3-30-ghz.html), here are the results:

```console
build$ bin/clang++ expresscall_benchmark.mm -o expresscall_benchmark -O3 -isysroot "$(xcrun --show-sdk-path)" -isystem "$(xcode-select -p)/CommandLineTools/usr/include/c++/v1" -isystem "$(xcode-select -p)/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1" -fmodules -fcxx-modules -framework Foundation
build$ ./expresscall_benchmark
Benchmarking average time to sum 1000000 numbers (over 100 trials)…
expresscall: 1965µs
objc_msgSend: 8817µs
direct: 615µs
swizzled expresscall: 6337µs
swizzled objc_msgSend: 5130µs
```

The results require some explanation, but it's clear that expresscall is significantly faster than anything but a call through a method marked with `__attribute__((objc_direct))`. The compiler has managed to inline the implementions for these two, so the time difference between them is the overhead to check whether the expresscall target is valid: one and a half microseconds per trial, which works out to about two-thirds of a nanosecond per method call. The swizzled `objc_msgSend` manages to take the fastest path through the function, coming out to two nanoseconds of overhead a call; the swizzled expresscall does the same but with the extra checks taking up the difference. The unswizzled `objc_msgSend` takes longer because the `count` and `numberAtIndex:` selectors end up colliding during method cache lookup, causing additional scanning to be necessary.

## Future extensions

expresscall easily outperforms `objc_msgSend` when its speculation pays off, which is ideally almost all the time. Even though it hasn't been particularly optimized, its branches are highly predictable and I expect that a modern CPU should be able to speculate its memory accesses ahead of time. As the current implementation does have a slight cost in terms of code size and memory usage, it may be worth changing the frontend implementation to selectively apply the expresscall optimization: perhaps it could be augmented with profile-guided optimization information. With additional runtime support, even the slow, swizzled case could be performed efficiently–after all, the checks we perform are exactly the ones that the top of `objc_msgSend` does, so we may be able to skip some of them. Finally, for extra space savings the checks could be factored out into their own, short function; if the benefits of inlining are not desired there may still be a performance benefit of pushing the speculated target on the stack prior to calling the checking function so it can conditionally tail call into that or `objc_msgSend`.
