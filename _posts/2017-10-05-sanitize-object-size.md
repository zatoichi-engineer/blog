---
layout: post
title: "Sanitize Object Size and Its Performance Impact"
date: 2017-10-05
---
### Sanitizing memory accesses based on object sizes

Building on the previous post which covered
[stack smashing protection]({% post_url 2017-10-04-stack-smashing-protection %}),
this post covers the "hardening" option in GCC for sanitizing memory
accesses based on the size of objects.

At compile time the size of certain objects is known. GCC provides limited
buffer overflow protection which can detect and prevent certain out-of-bounds
read and writes based on the known object size. Code can determine this
size by using the built-in
[`__builtin_object_size()`](https://gcc.gnu.org/onlinedocs/gcc/Object-Size-Checking.html)
function. In addition, GCC also provides a means for sanitizing such object
accesses at runtime by instrumenting code to check that object accesses
are valid.

The topic of this post is GCC's `-fsanitize=object-size` flag. This flag
will add instrumentation around object accesses when the object size is
known at compile time. If an out-of-bounds access is detected, the program
will emit a warning (but continue to run). Consider the following example
program:

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void attemptAccess(int *a)
{
  printf("%d\n", a[4]);
}

int main(void)
{
  int a[4] = {1, 2, 3, 4};

  // caught by -fsanitize=bounds and -fsanitize=object-size
  for (size_t i = 0; i <= sizeof(a) / sizeof(a[0]); i++)
  {
    printf("%d\n", a[i]);
  }

  // caught by -fsanitize=object-size (with -O1 or -O2)
  int *b = a;
  for (size_t i = 0; i <= sizeof(a) / sizeof(a[0]); i++)
  {
     printf("%d\n", b[i]);
  }

  // caught by -fsanitize=object-size (with -O2)
  attemptAccess(a);

  // Get size from somewhere else so it will not be
  // known at compile time.
  int length = atoi(getenv("LENGTH"));
  char buffer[length];
  int index = 0;
  while(index < length)
  {
    buffer[index++] = '\0';
  }

  printf("Loading past the buffer\n");
  // caught by -fsanitize=bounds, not -fsanitize=object-size
  buffer[index] = '\0';

  return 0;
}
{% endhighlight %}

When compiled with `-fsanitize=object-size` and run, it will detect the
invalid accesses and print out a helpful message using libubsan:

{% highlight shell_session %}
$ gcc test.c -fsanitize=object-size -O2
$ LENGTH=4 ./a.out
1
2
3
4
test.c:17:5: runtime error: load of address 0x7ffe9add55e0 with insufficient space for an object of type 'int'
0x7ffe9add55e0: note: pointer points here
 04 00 00 00  00 00 00 00 00 00 00 00  00 19 99 9a 38 87 44 4a  00 0a 40 00 00 00 00 00  00 00 00 00
              ^
0
1
2
3
4
test.c:24:5: runtime error: load of address 0x7ffe9add55e0 with insufficient space for an object of type 'int'
0x7ffe9add55e0: note: pointer points here
 04 00 00 00  00 00 00 00 00 00 00 00  00 19 99 9a 38 87 44 4a  00 0a 40 00 00 00 00 00  00 00 00 00
              ^
0
test.c:7:3: runtime error: load of address 0x7ffe9add55e0 with insufficient space for an object of type 'int'
0x7ffe9add55e0: note: pointer points here
 04 00 00 00  00 00 00 00 00 00 00 00  00 19 99 9a 38 87 44 4a  00 0a 40 00 00 00 00 00  00 00 00 00
              ^
0
Loading past the buffer
{% endhighlight %}

This behavior is useful for locating and fixing issues during development
or testing. However, because only a message is printed, if a bug makes it
through development and QA and into customer's hands it could leave a
product vulnerable to an exploit.

Much how stack smashing protection can halt a program to reduce an exploit
to a denial-of-service issue, GCC can cause a program to terminate when
an object is accessed out-of-bounds. To do this, the following flags
need to be added:

`-fno-sanitize-recover`:
Cause the program to stop execution when an issue is reported. The helpful
message is still printed using libubsan. The program exits normally with
a result of 0.

`-fsanitize-undefined-trap-on-error`:
When the program hits a sanitization issue it will terminate
by invoking __builtin_trap() and not use libubsan to print a message.

The first flag is necessary to cause the program to terminate. It still
requires using libubsan, however. The program also terminates politely, so
it may not be easy to detect in a production system. The second flag makes
the failure harder to ignore, and additionally does not require libubsan
which can reduce the number of dependencies and code storage space required.

{% highlight shell_session %}
$ gcc test.c -fsanitize=object-size -fno-sanitize-recover -fsanitize-undefined-trap-on-error -O2
$ LENGTH=4 ./a.out
1
2
3
4
Illegal instruction
{% endhighlight %}

### Analyzing the object size sanitizer

The object-size sanitizer is analyzed below. As the goal is for issues to cause
hard failures, sanitizer failures were configured to trap on an error. Two
metrics relevant to an embedded system will be used for the analysis:

1. Increased code size
2. Performance cost

To facilitate the analysis, a custom Linux distribution was built using Yocto,
one build with the object-size sanitizer enabled and one without. The build was
run on QEMU and analyzed. See
[this post]({% post_url 2017-10-02-yocto-on-osx %}#building-with-yocto-on-docker)
on how to create a custom QEMU image, in my case on macOS.

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

# Only build with the object-size sanitizer and ignore all other security flags.
SECURITY_CFLAGS = "-fsanitize=object-size -fno-sanitize-recover -fsanitize-undefined-trap-on-error"
SECURITY_NO_PIE_CFLAGS = "-fsanitize=object-size -fno-sanitize-recover -fsanitize-undefined-trap-on-error"
SECURITY_LDFLAGS = ""
SECURITY_X_LDFLAGS = ""
{% endhighlight %}


#### Code size

The Yocto builds were configured to produce a EXT4 file system image.
Following are the sizes of the file systems of the two builds:

| Build         | Size (KB)       
| ------------- |:-------------:|
| No Flags      | 556,866       |
| Sanitizer     | 557,376       |

This shows that SSP code instrumentation adds an additional 510 KB (~0.5MB)
of storage, which is an increase of ~0.09%. Your mileage may vary, as the increase
depends on the type of code being compiled.

#### Performance cost

Adding the additional instructions will result in some performance cost,
as the additional instruction do take a non-zero amount of time to execute.
However, the question is can the performance cost be measured or is it
exceedingly small?

To quantify the performance impact, an experiment was conducted which encoded
a small video using FFmpeg 20 times in succession, once with and once without
the sanitizing flags. See [this post]({% post_url 2017-10-04-stack-smashing-protection %}#performance-cost) for details on the experiment and the video file which
was used.

The following two [box plots](http://www.physics.csbsju.edu/stats/box2.html)
show the results of the experiment (raw data [here]({{ "/assets/data/hardening-ffmpeg-data.csv" | absolute_url }})).

![Boxplot]({{ "/assets/images/sanitize-object-size-ffmpeg-boxplot.png" | absolute_url }})

The results show that there is a clear increase in the amount of time needed
to encode the video with the sanitizer enabled. On average the sanitizer resulted
in an increase of 12 seconds, or ~10%.

### Conclusion

The object-size sanitizer does provide some protection against out-of-bounds
accesses if GCC can determine the size of objects at compile time. Using it
does increase the size of executables modestly, but comes with a significant
performance penalty (~10% in this case). The sanitizer would be valuable
during development and QA, but may not be recommended for systems where
performance targets may be missed. There are some projects where this
sanitizer is enabled as the security trade-off is worth the performance hit
(an example is the
[CopperheadOS](https://copperhead.co/android/docs/technical_overview)
Android distribution), however the trade-off may not be worth it for many
projects.
