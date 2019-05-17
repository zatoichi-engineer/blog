---
layout: post
title: "FORTIFY_SOURCE and Its Performance Impact"
date: 2017-10-06
---
### Buffer overflow checking in glibc

The last few topics discussed "hardening" options in GCC which
instrumented code, checking at runtime for certain undefined behavior
regarding memory read or writes. This post takes a look at a different
hardening option unrelated to instrumenting code, but instead at
bounds checking in glibc.

There are several functions in glibc which operate on strings or buffers.
These include:

memcpy, mempcpy, memmove, memset, strcpy, stpcpy, strncpy, strcat,
strncat, sprintf, vsprintf, snprintf, vsnprintf, gets

Although some of them accept an argument defining the length of the
passed buffer, the functions must believe that the size is correct.
Other functions, such as sprintf, do not accept a maximum size argument.
In both of these cases, it is possible for a buffer overflow to occur.

Glibc supports several wrapper functions for these which do perform bounds
checking at runtime on the actual size of the buffers being processed. For example,
 [`__memcpy_chk()`](http://refspecs.linux-foundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/libc---memcpy-chk-1.html)
supports an extra argument for the size of the destination buffer:

{% highlight c %}
 __memcpy_chk(void * dest, const void * src, size_t len, size_t destlen)
{% endhighlight %}

These wrappers function in the same way that the original function do, with the
exception that if the destination buffer will not accommodate the data being
written the function aborts at runtime, thus pointing out a buffer overflow before
it occurs.

One is not expected to call these wrapper functions directly. Instead, if
code is compiled with the `_FORTIFY_SOURCE` macro defined GCC will transparently
replace the function calls with their wrappers if the compiler cannot prove
that a buffer overflow is impossible. The size of the destination
buffer is determined using the built-in function `__builtin_object_size()`.
This returns the bytes remaining in a structure; if the size is not
known at compile time `(size_t) -1 ` is used (and thus no additional protection
is provided).

The `__FORITY_SOURCE` macro can be configured to check in two different modes,
as the definition of "bytes remaining in a structure" has one of two possible
meanings. Consider the following structure:

{% highlight c %}
struct blob
{
   char first[10];
   char second[10];
};
{% endhighlight %}

The blob.first substructure has only 10 bytes. If writing
past this and into blob.second should be disallowed, use `_FORTIFY_SOURCE=2`.
Alternatively, if data overruns from blob.first into blob.second, but does not
overrun blob, the program may still work correctly (and this may be part of
the program's expected operation). If this is acceptable, use `_FORTIFY_SOURCE=1`.
To summarize:

{% highlight c %}
struct blob b;
// Allowed in both
memset(b.first, 0, sizeof(b.first));
// Allowed in _FORTIFY_SOURCE=1 only
memset(b.first, 0, sizeof(b));
// Not allowed with either
memset(b.first, 0, sizeof(b)+1);
{% endhighlight %}

In addition to checking at runtime, if the compiler knows at compile time that a
function call will cause a buffer overflow it will also emit a warning. For
example:

{% highlight c %}
#include <string.h>

int main()
{
   char buffer[5];
   memset(buffer, 0, sizeof(buffer)+1);
   return 0;
}
{% endhighlight %}

{% highlight shell_session %}
$ gcc -D_FORTIFY_SOURCE=2 -O2 test.c
In file included from /usr/include/string.h:635:0,
                 from test.c:1:
In function ‘memset’,
    inlined from ‘main’ at test.c:6:4:
/usr/include/x86_64-linux-gnu/bits/string3.h:90:10: warning: call to __builtin___memset_chk will always overflow destination buffer
   return __builtin___memset_chk (__dest, __ch, __len, __bos0 (__dest));
{% endhighlight %}

Unless compiled with -Werror the program will still compile. However, when
run it will fail:

{% highlight shell_session %}
$ ./a.out
*** buffer overflow detected ***: ./a.out terminated
======= Backtrace: =========
/lib/x86_64-linux-gnu/libc.so.6(+0x777e5)[0x7fae95ae97e5]
/lib/x86_64-linux-gnu/libc.so.6(__fortify_fail+0x5c)[0x7fae95b8b11c]
/lib/x86_64-linux-gnu/libc.so.6(+0x117120)[0x7fae95b89120]
./a.out[0x4004e8]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0)[0x7fae95a92830]
./a.out[0x400539]
======= Memory map: ========
00400000-00401000 r-xp 00000000 08:01 928752                             /workdir/build/a.out
00600000-00601000 r--p 00000000 08:01 928752                             /workdir/build/a.out
00601000-00602000 rw-p 00001000 08:01 928752                             /workdir/build/a.out
02588000-025a9000 rw-p 00000000 00:00 0                                  [heap]
7fae9585c000-7fae95872000 r-xp 00000000 08:01 918443                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7fae95872000-7fae95a71000 ---p 00016000 08:01 918443                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7fae95a71000-7fae95a72000 rw-p 00015000 08:01 918443                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7fae95a72000-7fae95c32000 r-xp 00000000 08:01 918422                     /lib/x86_64-linux-gnu/libc-2.23.so
7fae95c32000-7fae95e32000 ---p 001c0000 08:01 918422                     /lib/x86_64-linux-gnu/libc-2.23.so
7fae95e32000-7fae95e36000 r--p 001c0000 08:01 918422                     /lib/x86_64-linux-gnu/libc-2.23.so
7fae95e36000-7fae95e38000 rw-p 001c4000 08:01 918422                     /lib/x86_64-linux-gnu/libc-2.23.so
7fae95e38000-7fae95e3c000 rw-p 00000000 00:00 0
7fae95e3c000-7fae95e62000 r-xp 00000000 08:01 918402                     /lib/x86_64-linux-gnu/ld-2.23.so
7fae96056000-7fae96059000 rw-p 00000000 00:00 0
7fae9605e000-7fae96061000 rw-p 00000000 00:00 0
7fae96061000-7fae96062000 r--p 00025000 08:01 918402                     /lib/x86_64-linux-gnu/ld-2.23.so
7fae96062000-7fae96063000 rw-p 00026000 08:01 918402                     /lib/x86_64-linux-gnu/ld-2.23.so
7fae96063000-7fae96064000 rw-p 00000000 00:00 0
7ffdbdcab000-7ffdbdccc000 rw-p 00000000 00:00 0                          [stack]
7ffdbdd99000-7ffdbdd9b000 r--p 00000000 00:00 0                          [vvar]
7ffdbdd9b000-7ffdbdd9d000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
Aborted
{% endhighlight %}

Using `_FORTIFY_SOURCE` is not a panacea, as not all calls to these functions can be checked
for overflows. Below shows examples of the four possible cases
 [[source](https://gcc.gnu.org/ml/gcc-patches/2004-09/msg02055.html)]:

{% highlight c %}
char buf[5];

/*
 * 1) Known correct.
 * No runtime checking is needed, memcpy/strcpy
 * functions are called (or their equivalents inline).
 */
memcpy (buf, foo, 5);
strcpy (buf, "1234");

/*
 * 2) Not known if correct, but checkable at runtime.
 * The compiler knows the number of bytes remaining in object,
 * but doesn't know the length of the actual copy that will happen.
 * Alternative functions __memcpy_chk or __strcpy_chk are used in
 * this case that check whether buffer overflow happened. If buffer
 * overflow is detected, __chk_fail () is called (the normal action
 * is to abort () the application, perhaps by writing some message
 * to stderr.
 */
memcpy (buf, foo, n);
strcpy (buf, bar);

/*
 * 3) Known incorrect.
 * The compiler can detect buffer overflows at compile
 * time. It issues warnings and calls the checking alternatives
 * at runtime.
 */
memcpy (buf, foo, 6);
strcpy (buf, "12345");

/*
 * 4) Not known if correct, not checkable at runtime.
 * The compiler doesn't know the buffer size, no checking
 * is done.  Overflows will go undetected in these cases.
 */
memcpy (p, q, n);
strcpy (p, q);
{% endhighlight %}

Using `_FORTIFY_SOURCE` also adds compile time warnings and checks
for some best-practices. Read [here](https://wiki.ubuntu.com/ToolChain/CompilerFlags#A-D_FORTIFY_SOURCE.3D2) for further details.

### Analyzing FORTIFY_SOURCE

The usage of `_FORTIFY_SOURCE` is analyzed below. The most strict option,
`_FORTIFY_SOURCE=2`, is used. Two metrics relevant to an embedded system will be
used for the analysis:

1. Increased code size
2. Performance cost

To facilitate the analysis, a custom Linux distribution was built using Yocto,
one build with the define enabled and one without. The build was
run on QEMU and analyzed. See
[this post]({% post_url 2017-10-02-yocto-on-osx %}#building-with-yocto-on-docker)
on how to create a custom QEMU image, in my case on macOS

The Yocto build was a bare-bones build with one exception: FFmpeg was included
which will be used to compare performance. Adding FFmpeg was accomplished by
adding the following to the conf/local.conf file in the build directory:

{% highlight shell %}
# Add ffmpeg and allow its commercial license flag
CORE_IMAGE_EXTRA_INSTALL += "ffmpeg"
LICENSE_FLAGS_WHITELIST += "commercial"

# Add extra space (in KB) to the file system, so that the
# benchmark has space to output its file(s).
IMAGE_ROOTFS_EXTRA_SPACE = "512000"
{% endhighlight %}

To enable the sanitizer the following was added to the conf/local.conf file:

{% highlight shell %}
# Include the following file:
#    poky/meta/conf/distro/include/security_flags.inc
# which enables all packages to build with additional
# security flags. Some packages cannot be built with
# some flags, so blacklist those.
require conf/distro/include/security_flags.inc

# Only build with the define and ignore all other security flags.
SECURITY_CFLAGS = "-D_FORTIFY_SOURCE=2"
SECURITY_NO_PIE_CFLAGS = "-D_FORTIFY_SOURCE=2"
SECURITY_LDFLAGS = ""
SECURITY_X_LDFLAGS = ""
{% endhighlight %}

#### Code size

The Yocto builds were configured to produce a EXT4 file system image.
Following are the number of KB used on the file systems:

| Build          | Size (KB)       
| -------------- |:-------------:|
| No Flags       | 34,844       |
| FORTIFY_SOURCE | 34,868       |

This shows that compiling with `_FORTIFY_SOURCE=2` results in adding an
additional 24 KB, which is very modest. This may depend on the type of
code being compiled, so your mileage may vary.

#### Performance cost

The wrappers for the fortified glibc functions do result in extra instructions
being executed, checking bounds conditions before continuing on.
However, the question is can the performance cost be measured or is it
exceedingly small?

To quantify the performance impact, an experiment was conducted which encoded
a small video 20 times in succession using FFmpeg, once with and once without
using the fortified functions. See [this post]({% post_url 2017-10-04-stack-smashing-protection %}#performance-cost) for details on the experiment and
the video file which was used.

The following two [box plots](http://www.physics.csbsju.edu/stats/box2.html)
show the results of the experiment (raw data [here]({{ "/assets/data/hardening-ffmpeg-data.csv" | absolute_url }})).

![Boxplot]({{ "/assets/images/fortify-source-ffmpeg-boxplot.png" | absolute_url }})

The results show that there is no loss of performance when using `_FORTIFY_SOURCE`.
Curious, the performance appears to have improved, which is not expected. It is
not known if this is an artifact of the testing setup or if there were additional
optimizations which became available when the option was used.

### Conclusion

The `_FORTIFY_SOURCE` option does provide protection against some types of
buffer overflows when using select glibc functions. The code size cost
when enabling this is very low, and there is no indication of performance
loss when encoding a sample video with FFmpeg. The option additionally will emit
compile time warnings when code can be proved offline to result in buffer overflows.
Given these results, enabling this option should be an simple and cheap way
to decrease the risk of buffer overflow related defects in software using
glibc.
