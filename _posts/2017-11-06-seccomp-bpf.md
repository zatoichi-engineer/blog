---
layout: post
title: "Seccomp-bpf and Its Performance Impact"
date: 2017-11-06
---
### Limiting available system calls

Effective software security is best done in layers; if an attacker is able
to penetrate one layer they encounter another. An attack strategy becoming
more common is to attack the kernel itself; if one succeeds in injecting
code into kernel space it is game over.

As an example of this, Ang Cui and Salvatore Solfo from Columbia University
discovered
[vulnerabilities in Cisco phones](https://arstechnica.com/information-technology/2013/01/hack-turns-the-cisco-phone-on-your-desk-into-a-remote-bugging-device/)
which allowed them to inject arbitrary code into kernel memory. They discovered
the necessary vulnerabilities by fuzzing the kernel by way of fuzzing
system calls.

Not all processes in a system need to be able to make all possible system calls.
Limiting the system calls possible has two positive outcomes:
- an attacker who exploits a vulnerability is able to do less with the
highjacked process
- fewer system calls are exposed, limiting access to potential vulnerabilities
in the kernel

One way to limit the system calls available to a process is by using
seccomp, short for Secure Computing Mode. Seccomp is a mechanism in the
Linux kernel which allows a process to make a one-way transition to a
secure mode where only exit(), sigreturn(), read(), and write() on
file descriptors already opened can be made. Any other system call attempted
will result in killing the process with a SIGKILL or SIGSYS.

An extension of this, [proposed in 2012](https://lwn.net/Articles/475043/),
is called seccomp-bpf, which allows the filtering of system calls using
Berkeley Packet Filter rules. It allows a process to more finely control
which system calls can be used, in addition to checking the arguments
passed to the system calls.

### Seccomp-bpf on ffmpeg

The seccomp-bpf technique is of interest, as it provides a flexible means
to specify allowed system calls on a per-process level. Past posts
used ffmpeg to compare performance of various security feature, and this
post will do so as well.

Through stracing ffmpeg all the system calls it uses to encode an example
video were found. The following are modifications to ffmpeg which
limit the available system calls to the minimum possible to successfully
do its job. The list of allowed calls is in the `sock_filter filter` structure.
Note that this syntax is adopted from
[this](https://outflux.net/teach-seccomp/) example.

{% highlight c %}
diff --git a/ffmpeg.c b/ffmpeg.c
index b26995d..e46a2c2 100644
--- a/ffmpeg.c
+++ b/ffmpeg.c
@@ -4298,11 +4298,120 @@ static void log_callback_null(void *ptr, int level, const char *fmt, va_list vl)
 {
 }

+#include <sys/prctl.h>
+#ifndef PR_SET_NO_NEW_PRIVS
+# define PR_SET_NO_NEW_PRIVS 38
+#endif
+
+#include <linux/unistd.h>
+#include <linux/audit.h>
+#include <linux/filter.h>
+#ifdef HAVE_LINUX_SECCOMP_H
+# include <linux/seccomp.h>
+#endif
+#ifndef SECCOMP_MODE_FILTER
+# define SECCOMP_MODE_FILTER   2 /* uses user-supplied filter. */
+# define SECCOMP_RET_KILL      0x00000000U /* kill the task immediately */
+# define SECCOMP_RET_TRAP      0x00030000U /* disallow and force a SIGSYS */
+# define SECCOMP_RET_ALLOW     0x7fff0000U /* allow */
+struct seccomp_data {
+    int nr;
+    __u32 arch;
+    __u64 instruction_pointer;
+    __u64 args[6];
+};
+#endif
+#ifndef SYS_SECCOMP
+# define SYS_SECCOMP 1
+#endif
+
+#define syscall_nr (offsetof(struct seccomp_data, nr))
+#define arch_nr (offsetof(struct seccomp_data, arch))
+
+#if defined(__i386__)
+# define REG_SYSCALL   REG_EAX
+# define ARCH_NR       AUDIT_ARCH_I386
+#elif defined(__x86_64__)
+# define REG_SYSCALL   REG_RAX
+# define ARCH_NR       AUDIT_ARCH_X86_64
+#else
+# warning "Platform does not support seccomp filter yet"
+# define REG_SYSCALL   0
+# define ARCH_NR       0
+#endif
+
+#define VALIDATE_ARCHITECTURE \
+       BPF_STMT(BPF_LD+BPF_W+BPF_ABS, arch_nr), \
+       BPF_JUMP(BPF_JMP+BPF_JEQ+BPF_K, ARCH_NR, 1, 0), \
+       BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_KILL)
+
+#define EXAMINE_SYSCALL \
+       BPF_STMT(BPF_LD+BPF_W+BPF_ABS, syscall_nr)
+
+#define ALLOW_SYSCALL(name) \
+       BPF_JUMP(BPF_JMP+BPF_JEQ+BPF_K, __NR_##name, 0, 1), \
+       BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_ALLOW)
+
+#define KILL_PROCESS \
+       BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_KILL)
+
+static void install_syscall_filter()
+{
+   struct sock_filter filter[] = {
+      /* Validate architecture. */
+      VALIDATE_ARCHITECTURE,
+      /* Grab the system call number. */
+      EXAMINE_SYSCALL,
+      /* List allowed syscalls. */
+      ALLOW_SYSCALL(rt_sigreturn),
+#ifdef __NR_sigreturn
+      ALLOW_SYSCALL(sigreturn),
+#endif
+      ALLOW_SYSCALL(exit_group),
+      ALLOW_SYSCALL(exit),
+      ALLOW_SYSCALL(open),
+      ALLOW_SYSCALL(close),
+      ALLOW_SYSCALL(read),
+      ALLOW_SYSCALL(write),
+      ALLOW_SYSCALL(ioctl),
+      ALLOW_SYSCALL(brk),
+      ALLOW_SYSCALL(rt_sigaction),
+      ALLOW_SYSCALL(fcntl64),
+      ALLOW_SYSCALL(fstat64),
+      ALLOW_SYSCALL(_llseek),
+      ALLOW_SYSCALL(mmap2),
+      ALLOW_SYSCALL(munmap),
+      ALLOW_SYSCALL(mremap),
+      ALLOW_SYSCALL(futex),
+      ALLOW_SYSCALL(getrusage),
+      ALLOW_SYSCALL(time),
+      ALLOW_SYSCALL(sched_getaffinity),
+      ALLOW_SYSCALL(_newselect),
+      ALLOW_SYSCALL(clock_gettime),
+      KILL_PROCESS,
+   };
+   struct sock_fprog prog = {
+      .len = (unsigned short)(sizeof(filter)/sizeof(filter[0])),
+      .filter = filter,
+   };
+
+   if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)) {
+      perror("prctl(NO_NEW_PRIVS)");
+      exit(1);
+   }
+   if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)) {
+      perror("prctl(SECCOMP)");
+      exit(1);
+   }
+}
+
 int main(int argc, char **argv)
 {
     int ret;
     int64_t ti;

+    install_syscall_filter();
+
     init_dynload();

     register_exit(ffmpeg_cleanup);
{% endhighlight %}

Configuring the list of allowed calls in BPF notation is rather cumbersome.
There are a few other alternatives which are simpler. First,
[libseccomp](https://github.com/seccomp/libseccomp) provides a function-call
based approach to setting up a filter. For example:

{% highlight c %}
#include <seccomp.h>

static void install_syscall_filter()
{
   scmp_filter_ctx ctx;
   int fd;
   ctx = seccomp_init(SCMP_ACT_TRAP);
   if (ctx == NULL)
      exit(1);

#define ADD_RULE(name) do{
   rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(name), 0); \
   if(rc < 0) \
      exit(1); \
   }while(0)

   ADD_RULE(rt_sigreturn);
   ADD_RULE(sigreturn);
   ADD_RULE(exit_group);
   ADD_RULE(exit);
   ADD_RULE(open);
   ADD_RULE(close);
   ADD_RULE(read);
   ADD_RULE(write);
   ADD_RULE(ioctl);
   ADD_RULE(brk);
   ADD_RULE(rt_sigaction);
   ADD_RULE(fcntl64);
   ADD_RULE(fstat64);
   ADD_RULE(_llseek);
   ADD_RULE(mmap2);
   ADD_RULE(munmap);
   ADD_RULE(mremap);
   ADD_RULE(futex);
   ADD_RULE(getrusage);
   ADD_RULE(time);
   ADD_RULE(sched_getaffinity);
   ADD_RULE(_newselect);
   ADD_RULE(clock_gettime);
#undef ADD_RULE

   rc = seccomp_load(ctx);
   if (rc < 0)
      exit(1);

   seccomp_release(ctx);
}
{% endhighlight %}

If using systemd and the process is defined by a service, the .service
file can define which system calls to allow with the
[SystemCallFilter](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#SystemCallFilter=)
option. As there are many system calls, systemd further defines several sets
of calls. For example, the following configuration should be sufficient
if running ffmpeg were a service:

{% highlight shell %}
SystemCallFilter="@basic-io @sync"
{% endhighlight %}

#### Performance cost

Adding checks for allowed system calls will result in some performance cost,
as running the filters will take a non-zero amount of time to execute.
However, the question is can the performance cost be measured or is it
exceedingly small?

To quantify the performance impact, two experiments were conducted. The
first encoded a small video 20 times in succession using FFmpeg, once with
and once without the filter installed mentioned in the diff earlier.
See [this post]({% post_url 2017-10-04-stack-smashing-protection %}#performance-cost)
for details on the experiment and the video file which was used.

The results from the first experiment are shown in the following two
[box plots](http://www.physics.csbsju.edu/stats/box2.html)
(raw data [here]({{ "/assets/data/hardening-ffmpeg-data.csv" | absolute_url }})).

![Boxplot]({{ "/assets/images/seccomp-ffmpeg-boxplot.png" | absolute_url }})

The results do not show an increase in the time necessary to encode the
example file. The reduction in mean amount of time may be a result of
a reduced number of samples in the experiment.

Unsatisfied with this result, a second experiment was run. In this experiment,
the same filters used in ffmpeg were optionally installed, then the following
system calls were attempted in a loop for 100,000 iterations:

{% highlight c %}
int fd = open("/dev/zero", O_RDONLY);
char data;
read(fd, &data, sizeof(data));
close(fd);
{% endhighlight %}

The duration of the resulting program was captured 100 times, both with and
without the filters installed. The following two box plots show the result
of this experiment:

![Boxplot]({{ "/assets/images/seccomp-open-read-close-boxplot.png" | absolute_url }})

This more clearly shows that there is a performance impact from using the filters.
On average there was a 0.58 second increase, which is 5.84 microseconds per
system call. This is an overhead of ~44% per call. However, as the much
larger experiment using ffmpeg did not show a noticeable difference I must
conclude that overall time of executing the system calls far exceeds the
added overhead.

### Conclusion

Enabling filters to prevent processes from executing some system calls is
a viable way to sandbox to a limited extent. Making changes on a per-process
basis may be cumbersome, as the source would need to be updated for each.
If the system uses systemd and the processes in question are services it
is easier to define the system call limits using configuration files.

The overhead for an simple system call may be relatively high, however the
overhead is easily hidden when the system calls take longer to execute.
As long as one can determine what system calls are valid for an application
using seccomp-bpf is a good approach for limiting one's risk and exposure.
