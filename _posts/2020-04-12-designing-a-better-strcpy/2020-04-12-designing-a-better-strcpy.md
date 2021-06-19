---
layout: post
title: "Designing a Better `strcpy`"
clean_title: "Designing a Better strcpy"
---

Like them or not, [null-terminated strings](https://en.wikipedia.org/wiki/Null-terminated_string) are essential to C, and working with them is necessary in all but the most trivial programs. While C-style strings are a fundamental part of using the language, manipulating them is a common source of security bugs and lost performance. One of the most common operations is copying a string from one buffer to another, and there are a variety of string functions that claim to do this in C. Anecdotally, however, there is much confusion about what they actually do, and many people desire a string copying function with the following properties:

1. The function should accept a null-terminated source string, a destination buffer, and an integer representing the size of the destination buffer.
2. Upon return the function should ensure that the destination buffer points to a null-terminated string containing a prefix of the source string when possible (specifically, when the destination buffer has a non-zero size) to avoid issues in the future with unterminated strings. (While string truncation has its own issues, it is often a fairly reasonable fallback.)
3. The function should indicate how many characters it copied from the source, as well as indicate if an overflow occurred. (This allows for dealing with the overflow, if desired.)
4. The function should be efficient, and it should not read or write memory that it does not have to. These go partially hand-in-hand: the function should run in a single pass, not write to the destination buffer past the NUL byte it places, or read characters from the source string once it's determined that it has filled the destination buffer. Ideally, the implementation would be vectorizable (relaxing some of the previous constraints slightly to within platform alignment guarantees).
5. The function should be standardized, so that it may be used portably across systems. Conformance to [ISO C](https://en.wikipedia.org/wiki/ANSI_C) or [POSIX.1](https://en.wikipedia.org/wiki/POSIX) are generally the most desirable.

That is, what is often necessary is the function below, which we'll call `strxcpy`:

```c
char *strxcpy(char *restrict dst, const char *restrict src, size_t len) {
	if (!len) {
		return NULL;
	}

	while (--len && (*dst++ = *src++))
		;

	if (!len) {
		*dst++ = '\0';
		return *src ? NULL : dst;
	} else {
		return dst;
	}
}
```

<!-- I made a mistake somewhere, didn't I? -->

Other than standardization, this function will copy the smaller of `strlen(src)` or `len - 1` bytes from `src` to `dst` and cap the copy with a NUL character. In the case where `src` fits in `dst`, it will return a pointer past the NUL byte it placed; otherwise it returns `NULL` to indicate a truncation. While [current](https://godbolt.org/z/f8nD4u) [compilers](https://godbolt.org/z/LDBm9y) seem to have trouble with its control flow, it should also be fairly straightforwards to vectorize, as the core loop is somewhat similar to a combination of `strncpy` and `strlen`.

With guidance to look back to, let's take a look at a variety of copying routines and see if they can help us.

{% include aside.html content="To head off the usual concerns, we'll assume that we must use C, and that we will be eschewing the various length-prefixed or aggregate string constructions available as third-party libraries. While using a different language can solve many of the issues in C besides the one mentioned here; it's not always desirable or even possible to utilize them. In addition to the usual drawbacks to using third-party libraries, replacing null-terminated strings often causes added syntactical overhead and incompatibilities with other code that has been designed to work with them." %}

## Some commonly used string copying routines

### `strcpy`

{% capture strcpy_summary %}
#### Signature

```c
#include <string.h>

char *strcpy(char *restrict dst, const char *restrict src);
```

#### Standardization

`strcpy` conforms to [ISO C90](https://en.cppreference.com/w/c/string/byte/strcpy).

#### Notes

The standard `strcpy` function, which copies characters from `src` to `dst`, up to and including the first NUL byte encountered. If `dst` is smaller than or aliases `src`, then the behavior of the program is undefined. `dst` is returned.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=strcpy_summary%}

`strcpy` certainly fulfills requirement 2 and parts of 4: it will always write out a null-terminated string and it'll do so quickly. However, it cannot perform bounds checks at all, so we can only use it if we know our source buffer is smaller than our destination buffer–it fails requirement 1. Plus it doesn't tell us how many characters it wrote, either–that's requirement 3. It's been part of C forever, so it does meet requirement 5.

### `strncpy`

{% capture strncpy_summary %}
### Signature

```c
#include <string.h>

char *strncpy(char *restrict dst, const char *restrict src, size_t len);
```

### Standardization

`strncpy` conforms to [ISO C90](https://en.cppreference.com/w/c/string/byte/strncpy).

### Notes

`strncpy` copies up to `len` characters from `src` to `dst`. If `src` is shorter than `len`, then `dst` is NUL-padded to `len` characters. `dst` is returned.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=strncpy_summary%}

`strncpy` takes the parameters we want, so it satisfies requirement 1; even in the face of an arbitrary source string it won't exhibit undefined behavior, provided that we supply it with the correct destination buffer length. However, if the source is longer that the destination, the buffer will not be null-terminated, and if it is shorter `strncpy` will continue writing NUL bytes to the destination up to its size. In addition, it doesn't indicate how many characters from the source were written, though it is possible to detect overflow by writing a NUL byte to the last character of the destination buffer and checking it after the call. That means it fails requirements 2, 3, and 4, but as it's been around in C for as long as `strcpy` it does meet requirement 5.

### `memcpy`

{% capture memcpy_summary %}
### Signature

```c
#include <string.h>

void *memcpy(void *restrict dst, const void *restrict src, size_t n);
```

### Standardization

`memcpy` conforms to [ISO C90](https://en.cppreference.com/w/c/string/byte/memcpy).

### Notes

Copies `n` bytes form `src` to `dst`, returning `dst`.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=memcpy_summary %}

`memcpy` doesn't care about NUL characters at all; it doesn't even require the source to be a null-terminated string. It fails the first three requirements right off the bat, but it's part of C and it sure is fast so it meets requirements 4 and 5.

### `strcpy_s`

{% capture strcpy_s_summary %}
### Signature

```c
#define __STDC_WANT_LIB_EXT1__ 1
#include <string.h>

#ifdef __STDC_LIB_EXT1__
errno_t strcpy_s(char *restrict dst, rsize_t len, const char *restrict src);
#endif
```

### Standardization

`strcpy_s` conforms to [ISO C11](https://en.cppreference.com/w/c/string/byte/strcpy), and is available if `__STDC_WANT_LIB_EXT1__` is defined prior to including string.h and `__STDC_LIB_EXT1__` is defined.

### Notes

A bounds-checked version of `strcpy` that performs the same operation and returns zero except it can write unspecified values to the remainder of `dst`, and if `src == NULL`, `dst == NULL`, if truncation would occur, `len` is zero or greater than `RSIZE_MAX`, or `src` and `dst` overlap, it will write a NUL byte to `*dst` if possible, return a nonzero value, and call a constraint handler function.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=strcpy_s_summary %}

On the surface, this function *seems* useful–but a closer look shows that it has a number of unfortunate issues. The largest is that any truncation will call a constraint handler function which can do many things, [like abort the program](https://en.cppreference.com/w/c/error/abort_handler_s). In addition, it doesn't tell us how much it wrote, can scribble over the destination, and is standardized but only available as an optional extension to C11. Overall, it only satisfies requirement 1.

### `strncpy_s`

{% capture strncpy_s_summary %}
### Signature

```c
#define __STDC_WANT_LIB_EXT1__ 1
#include <string.h>

#ifdef __STDC_LIB_EXT1__
errno_t strncpy_s(char *restrict dst, const char *restrict src, size_t len);
#endif
```

### Standardization

`strncpy_s` conforms to [ISO C11](https://en.cppreference.com/w/c/string/byte/strncpy), and is available if `__STDC_WANT_LIB_EXT1__` is defined prior to including string.h and `__STDC_LIB_EXT1__` is defined.

### Notes

A bounds-checked version of `strncpy`, that returns a non-zero value if `src == NULL`, `dst == NULL`, if truncation would occur, `len` is zero or greater than `RSIZE_MAX`, or `src` and `dst` overlap, in which case it will write the NUL byte to `*dst` if possible, unspecified values to the remainder of `dst`, return a nonzero value, and call a constraint handler function. Otherwise is will copy `len` bytes from `src` to `dst` and then add a NUL terminating byte at `dst[len - 1]`, returning zero.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=strncpy_s_summary %}

This function has the same constraint handler issue as `strcpy_s`, and is also standardized but often not available. While it will  null-terminate when the string fits and only clobber the destination on an error, it still only satisfies the first requirement.

### `stpncpy`

{% capture stpncpy_summary %}
#### Signature

```c
#include <string.h>

char *stpncpy(char *restrict dst, const char *restrict src, size_t len);
```

#### Standardization

`stpncpy` conforms to [POSIX.1-2008](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/string.h.html).

#### Notes

Identical to `strncpy`, except that a pointer to the written NUL byte is returned, if any; otherwise `dst + len` is returned.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=stpncpy_summary%}

`stpncpy` is an improvement on `strncpy`, but it only fixes the issue of detecting termination or overflow, which is requirement 3. It still fails requirement 2 because it doesn't necessarily null-terminate and it fails requirement 4 because it writes NULs to the end of the destination buffer. Unlike `strncpy` it's part of POSIX, but it still meets requirement 5 in addition to requirement 1.

### `snprintf`

{% capture snprintf_summary %}
#### Signature

```c
#include <stdio.h>

int snprintf(char *restrict dst, size_t len, const char *restrict fmt, ...);
```

#### Standardization

`snprintf` conforms to [ISO C99](https://en.cppreference.com/w/c/io/snprintf).

#### Notes

When used with `%s` as `fmt`, copies the first variadic parameter (a string) to `dst`, or the first `len - 1` bytes followed by the NUL byte. Returns the length of the first variadic parameter.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=snprintf_summary%}

`snprintf` is a somewhat strange inclusion, but it's a standard function that can help us if we use "%s" as the format string, taking a size and null-terminating its destination. It fulfills requirements 1, 2, and 5, but falls short on 3 and 4: its return value is essentially "what `sprintf` would have returned", which means it must perform an equivalent of a `strlen` at the very least. This is slow, not what we want, and an `int` (not a `size_t`).

### `strlcpy`

{% capture strlcpy_summary %}
#### Signature

```c
#include <string.h>

size_t strlcpy(char *restrict dst, const char *restrict src, size_t len);
```

#### Standardization

`strlcpy` is a common BSD extension.

#### Notes

Semantically equivalent to `sprintf(dst, len, "%s", src)` save for the return value, which is a `size_t`.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=strlcpy_summary%}

`strlcpy` is identical to the `sprintf` invocation from before, except it uses the correct `size_t` return type. This still means it fails to satisfy the performance requirements of 3 and 4, and it's not standard so it doesn't satisfy 5 either. Since it does the copy and leaves you with a null-terminated string it fills the first two requirements.

### `strscpy`

{% capture strscpy_summary %}
#### Signature

```c
size_t strscpy(char *restrict dst, const char *restrict src, size_t len);
```

#### Standardization

`strscpy` is a [Linux kernel function](https://www.kernel.org/doc/htmldocs/kernel-api/API-strscpy.html).

#### Notes

`strscpy` copies `src` to `dst` if it fits in the buffer and return the number of characters copied excluding the trailing NUL byte; otherwise it will copy the first `len - 1` characters and set `dst[len - 1]` to a NUL byte, returning `-E2BIG`.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=strscpy_summary%}

`strscpy` is the first function we've seen that satisfies the four functional requirements: it copies the as much of the source string as possible, null terminates the destination buffer, returns the number of characters copied, and does not perform excessive reads or writes. In fact, we can implement our `strxcpy` function using it:

```c
char *strxcpy(char *restrict dst, const char *restrict src, size_t len) {
	ssize_t copied = strscpy(dst, src, len);
	return copied != -E2BIG : src + copied + 1 : NULL;
}
```

It has two issues: the first, is that it returns an `ssize_t` rather than a `size_t`, but in practice this isn't really a problem. The second is that it's unfortunately non-standard–it's something the Linux kernel wrote for itself–which means it violates requirement 5.

### `memccpy`

{% capture memccpy_summary %}
### Signature

```c
#include <string.h>

void *memccpy(void *restrict dst, const void *restrict src, int chr, size_t len);
```

### Standardization

`memccpy` is an [XSI extension](https://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap02.html#tag_02_01_04) to [POSIX.1-2001](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/string.h.html), and is [planned to be added](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2349.htm) to [ISO C2X](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2479.pdf).

### Notes

`memccpy` is identical to `memcpy`, but may stop prematurely if `src` contains `chr`, copying `chr` and returning pointer after its location in `dst`. If `len` characters are copied without encountering `chr`, then `NULL` is returned.
{% endcapture %}

{% include aside.html collapsible=true type="Summary" content=memccpy_summary %}

`memccpy`, when used with the NUL character, satisfies all the requirements except for the second one, but this is trivial to fix:

```c
char *strxcpy(char *restrict dst, const char *restrict src, size_t len) {
	char *end = memccpy(dst, src, '\0', len);
	if (!end) {
		dst[len - 1] = '\0';
	}
	return end;
}
```
While it'll ship in an upcoming C standard, it's already widely available as a popular, optional POSIX extension.

### Other functions

There's a couple of other functions–`stpcpy`, `mempcpy`, `sprintf`, `sprintf_s`, and `snprintf_s`–that have been omitted for brevity, as their behavior (and issues) are fairly self-explanatory based on the other functions. (`mempcpy` is a GNU extension.)

## Final thoughts

Copying strings in C is an extremely common operation, but doing so safely and efficiently is non-trivial. Almost all currently available string routines, standardized or not, have subtle quirks that often prevent them from matching the expectations of the programmer who reaches for them. This issue is compounded by the fact that many style guides or linters will recommend the use of one (or sometimes more than one!) of these functions to replace `strcpy` without discussing their limitations. Finally, as we saw above the functions compose quite poorly: our `strxcpy`, an academic but not improbable scenario, could not use any of them in its implementation; one can only imagine that those writing an ad-hoc replacement for it may make both errors in doing so.

In contrast, the standardization of `memccpy` is a very welcome improvement, as it facilitates the construction of safer and more efficient string algorithms–in addition to `strxcpy`, a number of the functions discussed are *also* easy to construct with it. As it becomes more widespread, most code that relies on some of the semantics of `strxcpy` but uses one of the other functions to achieve it should probably migrate to `memccpy`, and ideally the push to phase out the use of them will drive the standardization and adoption of many more widely applicable string functions.
