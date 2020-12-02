## Welcome to Gray Area: Hackers Notes

Below is a place where I will keep some notes related to hacking, any courses I take, and any research that I do. I'll try to be good about keeping this updated, but if it ever gets out of date, please open an issue and virtually punch me.

## Simple AF network fuzzing with decept and mutiny

Network fuzzing is easy af. I had put off learning how to do it well, I knew the basics and had used BooFuzz and Peach before, and I was honestly just dreading creating a grammar for each one. For thoroughness I should probably do this using Boo - especially for stuff like HTTP [RFC-2616](https://tools.ietf.org/html/rfc2616) we should be able to build a list of all special HTTP headers that may have some special processing functions within an executable that could use some fuzzing. But damn, that would take like... over an hour so f\*ck that. Also according to fuzzguru Richard, and of course this is true but I never thought about it much, mutational fuzzing is often far more effective than generational fuzzing. 

We're on an exercise break now where we're finding a known bug in vshttpd. I don't like finding known things in known places so I'm going ahead and installing a variety of web servers, throwing some packets their way, and fuzzing them using [Decept Proxy](https://github.com/Cisco-Talos/Decept) and the associated [mutiny fuzzer](https://github.com/Cisco-Talos/mutiny-fuzzer) to just get things up and running first. I can fine tune shit later (another recommendation from Richard). Here's how to use decept:

`python decept.py --fuzzer apache2.fuzzer 127.0.0.1 9999 192.168.180.129 80`

All I'm saying there is listen on port 9999 on 127.0.0.1 and send to port 80 on 192.168.180.129, my NAT IP address then save the packet information to apache2.fuzzer, a file with a special grammar similar to one seen in BooFuzz and such that tells it where to fuzz. Decept is nice, it has no dependencies and is super easy to use. Since Decept and Mutiny are meant to be used together we get the .fuzzer file to pass in to Mutiny. This is also a simple bash one-liner

`python mutiny.py -i 192.168.180.129 apache2.fuzzer`

Note that the fuzzfile for now, because it is simply using decept proxy is actually going to fuzz the CONNECT verb here, for better coverage, we should probably generate more traffic, but I leave that as an exercise to the reader (read: go f*** yourself).

Once stuff is up and running you should see:

\*\*\*\*\*\*\*\*\*\*\The Mutiny Fuzzing Framework\*\*\*\*\*\*\*\*\*\*\
jaspadar@cisco.com && liwyatt@cisco.com <(^_^)>|
Monitor locking execution till condition met\
[*] Target detected, fuzzing.\
Message 0, seed: 3380, old len: 197, new len 230\
Message 0, seed: 3381, old len: 197, new len 328\
Message 0, seed: 3382, old len: 197, new len 216\
Message 0, seed: 3383, old len: 197, new len 230\
Message 0, seed: 3385, old len: 197, new len 195\
Message 0, seed: 3386, old len: 197, new len 203


Mutiny will start Mutiny-ing. Boom, we've network fuzzed. Note that mutiny/decept support a pretty decent set of options, you can MITM TLS and SSH traffic, create .fuzzer files from pcaps, and override a bunch of stuff. This is just the basics, but really you can extend the above concept quite a bit to fuzz a bunch of stuff quite quickly. I started fuzzing 3 web servers in about 5-10 minutes (and then wrote this up in another 5). That's pretty damn good.


## Hongfuzz vs. Apache httpd - FIGHT

Hi All, in keeping with the theme of quick iterative notes on wtf I'm up to here is how to get Hongfuzz up and running against apache http. The creators of Honggfuzz have wisely and kindly created a process for fuzzing. That means a lot of people have probably done it and it's a "known" technique. However, I've always found that I think that about any 0-day hunting - "eh, someone has probably looked, I won't spend my time on it". Well, that's how you miss bugs. Also not all fuzzers are deterministic, there's always the chance that I generate better inputs than the last a\*\*hole or that I have more CPU cores to leave it running for longer. Or maybe I'm wasting my time, but hey, then I know I wasted my time at least.

[So check this out](https://fossies.org/linux/honggfuzz/examples/apache/README.md). That's everything you need to get apache running under honggfuzz. Some quick notes on it: download the libs and build them, only use apt for the dependent libraries, you're going to need to build them all and enter the path in several config files, so put your sh\*t somewhere easy to find. I've noticed that there's been an update to honggfuzz and an additional parameter was needed for me to get this running. Below is the full command:

`../honggfuzz/honggfuzz -i corpus_http1/ -w httpd-2.4.46/httpd.wordlist --threads 10 -P -- ./bin/httpd -DFOREGROUND -f conf/httpd.conf.h1`


YDMV (your directories may vary), but note the -P added here. This tells it to run in persistent mode. I'm not convinced this is working well yet, and I'll be updating this post as I play with some of the options a bit more, but at least it runs. I'm getting way more timeouts than is reasonable (i'm at several hundred in just a few minutes), but at least no immediate crashes, which is expected on a hardened codebase like Apache httpds. I'll keep you all posted on results!

## So I went hunting for databases....

Huge shock, I found about 20+ from 2020 and another 20+ from 2019. This brings our total databases in scylla to about 290-300. and about 4.1 TB of data in JSON lines, one record per line. That's a lotta data. Honestly is it even worth learning how to hack anymore?? Just kidding, I live for this shit. Anyway check out scylla.sh over the next few days for the data.


## Sweet sweet sysinternals

So I'm going to go through every sysinternals tool that I deemed "interesting" for 0-day hunting in this post. But don't take my word for it (right Levar Burton!?), look at sysinternals tools for yourself and go through them. I'll be going through them one by one here, but nothing is better than hands on experience using them for 0-day hunting.

I'll start off with some notes here that I'm making about memory changes in the various Windows versions via sysinternal's vmmap. For those of you that don't know I have a project called scylla.sh at https://scylla.sh where I work with very very large (sometimes TB large) file sizes. I primarily use Linux, but it isn't odd that I find a site where I need to use Windows to get it to actually work (the places I go to hunt for free DBs are... interesting). Anyway, point is, sometimes I'm writing python scripts on Windows and when you're talking trillions of lines and terabytes of data things get slooooow. One way to speed things up is by loading as much as I can to memory before performing operations against files. Anyway, you may have your own needs, so you might find the below specifics of Windows memory useful. Most of this information is form https://dzone.com/articles/windows-process-memory-usage-demystified and validated with Windows Internals Part I 7th edition. I'll summarize some of the more "interesting" (I have an odd definition of that) stuff I see below and not just write the exact same thing as that article I linked to.

One thing I learned that I frankly just did not know is that 32-bit virtual memory is limited to 2 GB whereas 64-bit memory is limited to 128TB of memory. That's a pretty huge difference and something to keep in mind when coding on large files.

Comitted vs. reserved memory is an important concept. Reserved mem is not guaranteed to be backed by physical memory (aka shit might not work yo), whereas reserved memory is backed by physical memory (shit will work yo). Committed memory can be shared by multiple processes. There exists a counter in the OS that counts the ammount of committed memory to ensure that processes don't end up using stuff that can't be backed physically. It's very reasonable to have reserved memory be way way higher than the actual physical memory to back it up- if a lot of processes are using a ton of memory then some shit needs to change on your OS (lookin' at you firefox - plug for what I still insist is a 0-day https://mega.nz/file/BmJFiCSa#Ga3wkyLkBE-LZdsgh-1gRCHwFoL9f3LsjUrkOnP9Kow that leads to simple as f*** ASLR bypass and pretty much getting around any memory protection you'd like on any flavor of Linux. According to firefox this was "not a vulnerability" and "detaching a debugger to a process doesn't constitute a vulnerability". Except, well, other processes don't allow it in Linux due to Ptrace protections, firefox does. This is a fun theme in 0-days I see all the time where something is "too simple" to be a 0-day. Hey you're just replacing a dll right? You're just able to arbitrarily read a procs memory right? you're just able to bypass ASLR, get stack canaries, determine readable/writable addresses, get password manager passwords, get system passwords (if run as root) right? Oh wait...

  But I digress. I'm talking about windows here so let's get back to sweet sweet memory and more of those sysinternals tools. Let's talk some more about memory, because it's important. Wtf is this? ![Unusable!?](https://imgur.com/fx770BG.png)  Is Windows really telling me that this memory is allocated to the process but isn't usable? wtf? why? Well, Windows memory management for large pages are 64kb aligned (more on pages later) so any memory allocated to a process that doesn't align with the 64kb boundary is lost forever :(. RIP in peace.
  
So some of you might be asking, why does all of this happen? Who's in charge here? Well, it's the massive ntoskrnl.exe file, the main kernel executable.  It's in charge of providing a core set of System services many of which are exposed via the win32 API or kernel devices (aka drivers). I guess it's time to talk a little more about memory pages. In Windows there's really two sizes of pages, from now on I'm only gonna talk about 64-bit Windows because that's the way the world is moving and Windows 10 doesn't come in 32-bit flavor anymore. So fuck it. Read some old books if you want to know more about x86 systems. Sorry I didn't mean to be rude, I'm just kind of tired, but really want to talk more about memory. This is sad. I know this.
  
Anyway, my sad life aside, 64bit processors support small and large pages of sizes 4kb and 2MB respectively. Of course a natural question is when is each one used? Larger pages save a lot of processor time because when seeking an address there's a thing called the translation look-aside buffer (TLB) that holds the addresses of various pages and their mapped hardware memory location and therefore a lot of memory addies. When doing something that requires a lot of memory it's much more efficient to have less entries in the TLB, thus yeah, 2 MB at a time instead of 4KB. Of course allocating that memory takes more, uh, memory, so that's why small pages exist, for memory-inexpensive operations. Remember we're in computer-math-land (base 2) so a kb (defined as 2^10 bytes) is 1024 bytes in base 2. As a side note, I still haven't gotten over the metric prefix kilo being used since the dawn of computers to mean 1024 instead of 1000, but whatever. Anyway, large pages are prone to failure because they have to be aligned on a large-page boundary - this can be thought of as, shit has to happen in 512 small page (4kb * 512 = 2MB) chunks yo so a large allocation can happen from pages 0 (yep we're in computer-land so we start at 0) to 511, or 512-1,023, but not at 550-1,073 for example. This means that it has to find continuous physical memory for it to allocate each page (note this does not mean each page has to be continuous with a previous page, just that memory used in a page must be continuous). This causes what we call fragmentation - remember defragmenting shit all the time back in the day? Yeah, that was just because of this inefficiency. Because memory limits are getting serious now with 128 TB able to be used, Windows 10 has the sense of Huge Pages, which are 1GB or 1024MB, they work very similar to how large pages vs small pages work, just with a bigger number. So to sum all of this confusing shit up, let's say I allocate a buffer `char *buf[1050000]` the OS plays a little game of "how do I get to 1 million 50 thousand bytes using my allowed chunk (page) sizes" and would allocate one large page, 2MB = 1024 kb = 1,048,576 bytes and then one small page = 4kb = 4096 bytes giving us a grand total of 1,052,672 bytes. Still with me? That means 1,050,000 - 1,048,576 = 1424 bytes. So another small page gets allocated, giving us a grand total of 1 large page, 2 small pages, and 2672 bytes left over that are now labeled as "unusable" if you're using vmmap to look at stuff.

So this has a fun side effect for us 0-day hunters. Say we're using ROP to exploit something (if you need a ROP primer check this out: https://blog.hyperiongray.com/pwning-dnsmasq-with-rop/, we need the gadgets in that code to be executable. But aren't the people that create programs and drivers and shit smart enough to make things read-only? After all that's the whole point of DEP right? Well, for efficiency, DEP protects *pages* of memory, meaning that if there's some code that needs to execute 4097 bytes worth of instructions and then have some data that is meant to be read-only, one of two things will happen: Windows is going to allocate 2 small pages and make both of those pages executable (DEP dead - we can use what is meant to be read-only data in our ROP chain) OR the program is going to crash. Which one is better? I'll leave that as an exercise to the reader. 

If you're still reading this, congrats, you're a nerd. So let's talk about paged vs non-paged pool memory in Windows 10. non-paged pool memory is used by stuff that *always definitely totally and absolutely* has a physical memory backing, in other words it's always in memory if your OS is running. This is usually used in things like kernel-mode drivers, IRPs to drivers, ISRs, DPCs, and DIRQLs. Did I just start speaking in tongues? Ancient Aramaic!?!? No, those are real things in Windows. IRPs are something request packets, basically double-words (4 bytes) that are used to ask device drivers politely from userland to do something. ISRs are Interrupt Service Requests. This is for stuff that does low-level I/O operations, meaning either the kernel or kernel drivers. These requests follow a typical process that's something like this conversation with the OS:

- Yo, I wanna like, stop you from doing I/O stuff for a sec bby, u down?
- If the OS is DTF things continue, otherwise this returns ASAP
- If the OS has Tinder swiped right, an ISR collects any information it needs to do it's thing and passes it to a Deferred Procedure Call
- A DPC has a DIRQL (essentially just a priority level) associated with it, stuff with higher DIRQLs are done first, this is added to the queue of DPCs
- An ISR returns as fast as possible

So in a way you can think of it as the OS procrastinating and saying "eh I'll do that later, I got this other thing to do first". There's some missed details in there to which I'll just point to MSDN for (https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/writing-an-isr) but that's a sort of rough outline.
 
So anyway, this low-levely stuff is the kind of stuff non-pooled memory has. This, along with a bunch of other reasons, is why kernel exploits often look a bit different than user-mode exploits and we should know what we're looking at. If you've ever thrown an exe into a disassembler you'll see in drivers that there's a bunch of Mutexes, IRPs, ISRs, DPCs, and a bunch of stuff like that being called. This is happening from *non-pooled memory* whereas when you're reversing a PE file you won't see any of that stuff or at least rarely see these things. This is because executables are working from pooled (I like to just call it "normal" memory) virtual memory and has no access to low level structures - it can only ask the kernel to do these things through IRPs. Just to drive these points home, let's take a look at this in a debugger, open up WinDBG (aside: it's come to my attention that most OGs call this Windbag, but I sorta like Win Debug better). Attach locally to your kernel and let's take a look at this:

```
lkd> !vm
Page File: \??\C:\pagefile.sys
  Current:   3014656 Kb  Free Space:   3014648 Kb
  Minimum:   3014656 Kb  Maximum:     50331648 Kb
Page File: \??\C:\swapfile.sys
  Current:     16384 Kb  Free Space:     16376 Kb
  Minimum:     16384 Kb  Maximum:     25164264 Kb
No Name for Paging File
  Current:  67107824 Kb  Free Space:  67107132 Kb
  Minimum:  67107824 Kb  Maximum:     67107824 Kb

Physical Memory:          4194044 (   16776176 Kb)
Available Pages:          3332119 (   13328476 Kb)
ResAvail Pages:           3959228 (   15836912 Kb)
Locked IO Pages:                0 (          0 Kb)
Free System PTEs:      4294985064 (17179940256 Kb)

******* 464832 kernel stack PTE allocations have failed ******


******* 459598336 kernel stack growth attempts have failed ******

Modified Pages:             26112 (     104448 Kb)
Modified PF Pages:          25658 (     102632 Kb)
Modified No Write Pages:       38 (        152 Kb)
NonPagedPool    0:            231 (        924 Kb)
NonPagedPoolNx  0:          17887 (      71548 Kb)
NonPagedPool    1:              8 (         32 Kb)
NonPagedPoolNx  1:          12233 (      48932 Kb)
NonPagedPool Usage:           239 (        956 Kb)
NonPagedPoolNx Usage:       30120 (     120480 Kb)
NonPagedPool Max:      4294967296 (17179869184 Kb)
PagedPool  0:               21987 (      87948 Kb)
PagedPool  1:               15796 (      63184 Kb)
PagedPool Usage:            37783 (     151132 Kb)
PagedPool Maximum:     4294967296 (17179869184 Kb)
Processor Commit:            2494 (       9976 Kb)
Session Commit:              2920 (      11680 Kb)
Shared Commit:              73067 (     292268 Kb)
Special Pool:                   0 (          0 Kb)
Kernel Stacks:              13104 (      52416 Kb)
Pages For MDLs:              2652 (      10608 Kb)
Pages For AWE:                  0 (          0 Kb)
NonPagedPool Commit:        63749 (     254996 Kb)
PagedPool Commit:           37783 (     151132 Kb)
Driver Commit:              11926 (      47704 Kb)
Boot Commit:                 4664 (      18656 Kb)
PFN Array Commit:           49697 (     198788 Kb)
System PageTables:            773 (       3092 Kb)
ProcessLockedFilePages:        10 (         40 Kb)
Pagefile Hash Pages:            0 (          0 Kb)
Sum System Commit:         262839 (    1051356 Kb)
Total Private:             541657 (    2166628 Kb)
Misc/Transient Commit:        683 (       2732 Kb)
Committed pages:           805179 (    3220716 Kb)
Commit limit:             4947708 (   19790832 Kb)
```

We can see a lot of what I talked about above. Let's take a look at the `Page File: \??\C:\swapfile.sys` line. You might be thinking "but Alex, you're talking about RAM, why tf is this talking about a FILE". Well, when your OS is running out of memory it starts to use disk space as RAM, this is hugely inefficient as, well, it's not loaded to memory but instead has to do stuff with file operations. This is really slow, so if you've ever had your OS slow to a crawl and it takes 10 minutes to click the x in firefox, you're likely page swapping (Linux does this as well btw).

Well this simple little post has turned into a whole thing. I'll continue down this rabbit hole tomorrow/whenever I have time!






 
 
 
 
  
  




