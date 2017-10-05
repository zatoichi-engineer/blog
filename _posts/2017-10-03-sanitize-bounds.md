---
layout: post
title: "Sanitize Bounds and Its Performance Impact"
date: 2017-10-03
---
### Sanitizing array accesses

Continuing on the theme from previous posts which covered different
"hardening" options in GCC this post covers a sanitizer for
out-of-bounds array accesses, enabled with the `-fsanitize=bounds` flag.

The GCC array bounds sanitizer instruments an executable to detect if
an array is being accessed out of bounds at runtime. Various invalid
accesses are detected. The checks are limited, however, as flexible
array members, flexible member-like arrays, and initializers of variables
with static storage are not checked
 [source](https://developers.redhat.com/blog/2014/10/16/gcc-undefined-behavior-sanitizer-ubsan/).

Following is a program which shows example issues that the bounds sanitizer is
able to check, and how it compares to the object-size sanitizer
[discussed earlier]({% post_url 2017-10-05-sanitize-object-size %}):

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void)
{
  int a[4] = {1, 2, 3, 4};

  // caught by -fsanitize=bounds and -fsanitize=object-size
  for (size_t i = 0; i <= sizeof(a) / sizeof(a[0]); i++)
  {
    printf("%d\n", a[i]);
  }

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

When compiled with `-fsanitize=bounds` and run, it will detect the
invalid accesses and print out a helpful message using libubsan:

{% highlight shell_session %}
$ gcc test.c -fsanitize=bounds
$ LENGTH=4 ./a.out
1
2
3
4
test.c:17:21: runtime error: index 4 out of bounds for type 'int [4]'
1
1
2
3
4
1
1
Loading past the buffer
test.c:42:9: runtime error: index 4 out of bounds for type 'char [*]'
{% endhighlight %}

This behavior is useful for locating and fixing issues during development
or testing. However, because only a message is printed, if a bug makes it
through development and QA and into customer's hands it could leave a
product vulnerable to an exploit.

As discussed in the [post]({% post_url 2017-10-05-sanitize-object-size %}) about
 `-fsanitize=object-size`, the `-fno-sanitize-recover` and
 `-fsanitize-undefined-trap-on-error` can be added to make hitting an issue
 fatal:

{% highlight shell_session %}
$ gcc test.c -fsanitize=bounds -fno-sanitize-recover -fsanitize-undefined-trap-on-error -O2
$ LENGTH=4 ./a.out
1
2
3
4
Illegal instruction
{% endhighlight %}

### Analyzing the bounds sanitizer

The bounds sanitizer is analyzed below. As the goal is for issues to cause
hard failures, sanitizer failures were configured to trap on an error. Two
metrics relevant to an embedded system will be used for the analysis:

1. Increased code size
2. Performance cost

To facilitate the analysis, a custom Linux distribution was built using Yocto,
one build with the bounds sanitizer enabled and one without. The build was
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

# Only build with the object-size sanitizer and ignore all other security flags.
SECURITY_CFLAGS = "-fsanitize=bounds -fno-sanitize-recover -fsanitize-undefined-trap-on-error"
SECURITY_NO_PIE_CFLAGS = "-fsanitize=bounds -fno-sanitize-recover -fsanitize-undefined-trap-on-error"
SECURITY_LDFLAGS = ""
SECURITY_X_LDFLAGS = ""
{% endhighlight %}

One further deviation was made to the build, as it was found that when the
bounds sanitizer was enabled FFmpeg hit an out-of-bounds array access
in libx264 (r2491, git hash
c8a773ebfca148ef04f5a60d42cbd7336af0baf6).

{% highlight shell_session %}
traps: ffmpeg[405] trap invalid opcode ip:43eca1c3 sp:bf9a2960 error:0 in libx264.so.144[43e86000+112000]
{% endhighlight %}

Initially FFmpeg was used with x264  git hash
. However, the sanitizer flag found
an out-of-bounds access during encoding.


#### Code size

The Yocto builds were configured to produce a EXT4 file system image.
Following are the sizes of the file systems of the two builds:

| Build         | Size (KB)       
| ------------- |:-------------:|
| No Flags      | 575,622       |
| Sanitizer     |        |

//This shows that SSP code instrumentation adds an additional 697 KB (~0.7MB)
//of storage, which is an increase of ~0.12%. Your mileage may vary, as the increase
//depends on the type of code being compiled.

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

![Boxplot]({{ "/assets/images/sanitize-bounds-ffmpeg-boxplot.png" | absolute_url }})

//The results show that there is a clear increase in the amount of time needed
//to encode the video with the sanitizer enabled. On average the sanitizer resulted
//in an increase of 11.5 seconds, or ~9.5%.

### Conclusion

The bounds sanitizer does provide some protection against out-of-bounds
array accesses at runtime. The coverage is not a subset of the object-size
sanitizer, as there are some issues that the bounds sanitizer catches that
the object-size sanitizer does not. Using the bounds sanitizer does increase
the size of executables
// modestly, but comes with a significant
//performance penalty (~9.5% in this case). The sanitizer would be valuable
//during development and QA, but may not be recommended for systems where
//performance targets may be missed. There are some projects where this
//sanitizer is enabled as the security trade-off is worth the performance hit
//(an example is the
//[CopperheadOS](https://copperhead.co/android/docs/technical_overview)
//Android distribution), however the trade-off may not be worth it for many
//projects.



(gdb) bt
#0  x264_slicetype_mb_cost (h=h@entry=0x80f4260, frames=0xbfcfcc18, p0=1, p1=1, b=1, dist_scale_factor=128, do_search=0xbfcfc830,
    w=0x4d956720 <x264_weight_none>, output_inter=0x8179fc0, output_intra=0x817a0ac, a=<optimized out>, a=<optimized out>)
    at /usr/src/debug/x264/r2491+gitAUTOINC+c8a773ebfc-r0/git/encoder/slicetype.c:782
#1  0x4d8b329b in x264_slicetype_slice_cost (s=s@entry=0xbfcfc8c0) at /usr/src/debug/x264/r2491+gitAUTOINC+c8a773ebfc-r0/git/encoder/slicetype.c:824
#2  0x4d8d8655 in x264_slicetype_frame_cost (h=h@entry=0x80f4260, a=a@entry=0xbfcfcc60, frames=frames@entry=0xbfcfcc18, p0=<optimized out>, p0@entry=1,
    p1=<optimized out>, b=<optimized out>, b_intra_penalty=<optimized out>)
    at /usr/src/debug/x264/r2491+gitAUTOINC+c8a773ebfc-r0/git/encoder/slicetype.c:936
#3  0x4d8db685 in x264_slicetype_decide (h=0x80f4260) at /usr/src/debug/x264/r2491+gitAUTOINC+c8a773ebfc-r0/git/encoder/slicetype.c:1802
#4  0x4d949a32 in x264_stack_align () from /workdir/sdk/sysroots/i586-poky-linux/usr/lib/libx264.so.144
#5  0x4d91c836 in x264_lookahead_get_frames (h=h@entry=0x80f4260) at /usr/src/debug/x264/r2491+gitAUTOINC+c8a773ebfc-r0/git/encoder/lookahead.c:233
#6  0x4d919771 in x264_encoder_encode (h=0x80f4260, pp_nal=<optimized out>, pi_nal=<optimized out>, pic_in=<optimized out>, pic_out=<optimized out>)
    at /usr/src/debug/x264/r2491+gitAUTOINC+c8a773ebfc-r0/git/encoder/encoder.c:3307
#7  0x4c921604 in X264_frame (ctx=0x80a7540, pkt=0xbfd0036c, frame=0x80f2840, got_packet=0xbfd00228)
    at /usr/src/debug/ffmpeg/3.1.3-r0/ffmpeg-3.1.3/libavcodec/libx264.c:336
#8  0x4cb0662c in avcodec_encode_video2 (avctx=<optimized out>, avpkt=<optimized out>, frame=<optimized out>, got_packet_ptr=<optimized out>)
    at /usr/src/debug/ffmpeg/3.1.3-r0/ffmpeg-3.1.3/libavcodec/utils.c:1962
#9  0x0806928a in do_video_out (s=0x808d0e0, ost=ost@entry=0x818fa00, next_picture=next_picture@entry=0x80f2840, sync_ipts=<optimized out>)
    at /usr/src/debug/ffmpeg/3.1.3-r0/ffmpeg-3.1.3/ffmpeg.c:1176
#10 0x0806b9f6 in reap_filters (flush=flush@entry=0) at /usr/src/debug/ffmpeg/3.1.3-r0/ffmpeg-3.1.3/ffmpeg.c:1367
#11 0x0804e96c in transcode_step () at /usr/src/debug/ffmpeg/3.1.3-r0/ffmpeg-3.1.3/ffmpeg.c:4118
#12 transcode () at /usr/src/debug/ffmpeg/3.1.3-r0/ffmpeg-3.1.3/ffmpeg.c:4162
#13 main (argc=<optimized out>, argv=<optimized out>) at /usr/src/debug/ffmpeg/3.1.3-r0/ffmpeg-3.1.3/ffmpeg.c:4357

fenc->lowres_costs[b-p0][p1-b][i_mb_xy] = X264_MIN( i_bcost, LOWRES_COST_MASK ) + (list_used << LOWRES_COST_SHIFT);

(gdb) print b
$25 = 1
(gdb) print p0
$26 = 1
(gdb) print p1
$27 = 1
(gdb) print b
$28 = 1
(gdb) print i_mb_xy
$29 = 919
(gdb) print fenc->lowres_costs
$30 = {{0xb6ba8f40, 0xb6ba9680, 0xb6ba9dc0, 0xb6baa500, 0xb6baac40, 0x0 <repeats 13 times>}, {0xb6bab380, 0xb6babac0, 0xb6bac200, 0xb6bac940,
    0xb6bad080, 0x0 <repeats 13 times>}, {0xb6bad7c0, 0xb6badf00, 0xb6bae640, 0xb6baed80, 0xb6baf4c0, 0x0 <repeats 13 times>}, {0xb6bafc00, 0xb6bb0340,
    0xb6bb0a80, 0xb6bb11c0, 0xb6bb1900, 0x0 <repeats 13 times>}, {0xb6bb2040, 0xb6bb2780, 0xb6bb2ec0, 0xb6bb3600, 0xb6bb3d40, 0x0 <repeats 13 times>}, {
    0x0 <repeats 18 times>} <repeats 13 times>}
(gdb) print sizeof(fenc->lowres_costs)
$31 = 1296
(gdb) print sizeof(uint16_t)
$32 = 2
