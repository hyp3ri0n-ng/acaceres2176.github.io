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

Well this simple little post has turned into a whole thing. I'll continue down this rabbit hole tomorrow/whenever I have time! OK picking this back up!

An interesting tidbit I'm learning more about is shareable memory, I noticed it in VMMAP:

![Types of memory](vmmap_memtypes.PNG)

and it seemed like an interesting attack surface. Shared memory means that two processes will use the same memory page, this is useful to save the very important limited physical memory (think about it - you have 128TB of addressable space *per process* and a solid modern laptop has something like 8-32GB of physical memory). But could this open up an attack vector where a not-important low privileged user has a process that has mapped a page to the same physical memory as a high privileged user with an important process with lots of cached goodies? Yep! And of course I'm not the first person to think of this (https://www.google.com/search?channel=tus2&client=firefox-b-1-d&q=shared+memory+attacks). Neat. So what are those other types of memory? Just for completion let's go through vmmap's categories. I don't like them, as they are different "layers" of categories, you'll see what I mean:

- Free - this is uhh, memory that is free and can be allocated.
- Heap - uh, memory used by heaps.
- Managed Heap - This is memory that's used by stuff where you don't have to explicitly define your heap usage (e.g. .NET in Windows manages its own memory unlike say, C++ or C)
- Mapped File - This is mostly memory used by processes, it is a type of shareable memory (interesting?)
- Stack - Memory used by stacks (more on stacks later)
- Shareable - This is memory that is allowed to be shared with different processes. 
- Page Table - This is private (unshareable) kernel physical memory that holds page table mappings
- Private Data - Private memory, not shareable between processes
- Unusable - we talked about this, due to memory management occuring in discrete chunks, some memory is wasted

And that's that. OK enough of VMMap, let's talk a little about it's physical brother RAMMap:

![RAMMap](rammap.PNG)
![RAMMap2-Electric-Boogaloo](rammap2.PNG

There's some overlap with VMMap but RAMMap talks mostly about physical memory. I've taken screenies of what I think are the most interesting tabs. Basically, they show a bunch of stuff about physical memory that are self explanatory. I've never used it in writing any exploits to be honest, it's still a little bit of black magic when people attack physical memory (like Rowhammer and such). These might come in useful, and I leave it as an exercise to the reader to  check this tool out. That's college professor speak for "go fuck yourself, I ain't explaining shit" so sorry about that.

OK, let's talk a bit about memory protections, CPU, and then let's relate all of this back to 0-day hunting and memory attacks (we'll be looking at a write-what-where). How is memory protected, there's a short but pretty powerful list of how memory is protected in Win10:

- What happens in kernel-mode stays in kernel-mode. Only kernel-mode instructions can access kernel pages. User processes can't.
- Remember we talked about virtual addressing? Think of it as NAT for memory - physical memory is never directly referenced, each process gets its own virtual address space. Processes can't touch other process' virtual addy space, so even if we have shared physical memory between two processes, there's no context with which to know this for each process. In theory.
- The windows API allows you to control page access type (read, write, execute). There's a bunch of low-level options to functions like VirtualProtect et al that allow you to control this. For example you can mark a page as read only, read-write only, execute only, etc.
- Remember all that talk about attacking shared memory? They thought of that. Shared mem section objects have ACLs to make sure that only the processes that created that shared memory can access that shared memory. That makes it so that you can't have another process with a more global context attempt to inform each process of shared memory. Each process is blind to the fact that the memory is shared, it's abstracted away for a reason. 

Simple huh? But it works. All of this talk about memory would be incomplete without talking a little about Data Execution Prevention (DEP) or NX on Linux (No Execute). The reason its called Data Execution Prevention is simple: each process is made up of two things, instructions that are meant to be executed and data that is usually operated on or read in some useful way by a process. Low-level snippets of assembly code are made up of what we call opcodes, two classic examples are 0xcc and 0x90, these are an interrupt and a nop (pronounced no-op, i.e. do nothing). Now let's look at some data: 0x41, 0x42. These are A and B in ascii encoding. What do you notice about the two? They're all just bytes! So how the fuck does a program know the difference? Standards! The PE (portable executable) format splits these things up into the data segment (holds data, like A and B) and the code segment that has the instructions that the CPU is going to run. Other than that - nothing differentiates them. We haven't talked about CPUs yet, but RIP is a CPU register (a little place where you can store a few bytes, in 64-bit land, 8 bytes aka 64 bits *gasp*) that points to (theoretically) somewhere in the code segment and executes instructions. But instructions are just series of bytes right (we call them opcodes)? And well, data is just a series of bytes too, so what if we were able to control where RIP is pointing to and put it somewhere in the data segment that we control (like an open file mapped to memory)? Alright, now we're talking exploit dev and 0-day hunting. As it turns out this is the premise of arbitrary code execution and the famous paper Smashing the Stack for Fun and Profit by Aleph1 and ruining everyone's con talk names by making them all fucking end in "For Fun and Profit" or some annoying jokey play on that. Alright so back to DEP, what this does is it marks the pages with data in them as non-executable, meaning if we point somewhere in the middle of the data, it's not just going to start thinking those are instructions and running them - it's going to throw a page fault and stop execution, log it, and generally ruin your day (and exploit). DEP is on by default in Win10.

Signing off here 12/7/2020 at 1:30 AMish.

## Fuzzing Interlude

As I was doing all of the above I realized I was ready to start some vulnerability hunting. We'll start with the basics and work our way into more and more complicated stuff. Kernel-land, despite having a lot of stuff to learn this is kinda random (64 byte aligned shit? wut?) is actually pretty simple once you've got the hang of it. The trick is to just look for some fairly simple patterns in assembly or a decompiler. But before we get into that, we want to maximize our hunting. If you've never seen my office I probably have circa 60-70 cores, most of them sitting there cold. So let's start fuzzing some stuff! The one trick I have for fuzzing is to google if something has ever been fuzzed. If it has, hit another part of the application. If it hasn't, hit the simplest possible part. One thing I haven't seen fuzzed that seems pretty complicated is explorer.exe's ability to open a variety of different files. Go ahead and try it, it's sorta like gnome-open or xdg-open in Linux, you point it at a file and if the extension is registered with a program it pops it open in whatever is most appropriate. This functionality seems like it'd need to know how to parse a lot of different things, find functionality, and then open the file in some manner. That's a few steps, and anything with more than a step or two IMHO is a decent target. So let's get down and dirty and fuzz it. I'll be using Binary Ninja, for one reason: Ilfak. Ilfak has never been particularly kind or nice whenever I buy an IDA license (I've bought like 3, totaling about 10k), one time he was downright rude when I asked when my download would come in, and third Ilfak your wallet pretty hard if we use IDA (wah wahh). So let's go with Binary Ninja and Ghidra, I'll be using the former for the wonderful lighthouse plugin and the latter for its decompiler if I feel like I need some extra help. As a debugger I'll be using x64 debug and kind of a new technique to me - I'm going to pop open explorer with a few files that I know it's going to recognize, step through the code with x64dbg, and see what lighthouse lights up. I'll know that this chain of functions (and their arguments!) are what make it do its thing, and I'll stop after everything is parsed but before explorer.exe pops some window open. I have no idea if this is actually going to work, but wtf why not, it's easier than trying to figure out what's going on with just a debugger and a decompiler. I've used lighthouse before for code coverage and fuzzing, but never for this, so we'll see if it's up to the task. I want to make sure I document my failures here and don't just leave you all with the impression that hacking is just things that work, failing hard is always an option.

Ok let's get cracking, I've fired open Binary Ninja, cmder, and I have lighthouse installed as a plugin to Binary Ninja along with bncov (never used it but seems like a nice alternative to lighthouse if I for some reason have issues), and x64dbg. Right off the bat x64dbg comes in very very handy with downloading all the pdbs I need. Simply go to the "Symbols" tab, check out the imports, highlight all of them, right click, and click download all pdbs. Then I watched We Were Soldiers with Mel Gibson. This is an important step because symbols will take a while to download. Man, Vietnam was fucking stupid, fucking Lyndon Johnson am I right?? Anyway, back to hacking. I'm  going to need DynamoRio and a few iterations of running explorer.exe, then I'll see which functions pop out at me. It should be pretty easy to find the parsing function, and since I have symbols, things get even easier. So let's try it out. I ~~put on my robe and wizards hat~~ download DynamoRIO and get ready to use drcov. It's pretty simple to use, here's an example:

`drrun.exe -t drcov -- explorer.exe folder/`

This will pop open a folder, once again explorer has done some magic I don't yet understand, determined that I've passed it a folder, and opened the file explorer with that folder as the target. Again just complicated enough that there could be something juicy there. Let's try it out. But first things first, we need to know if explorer.exe being used by the system is 32 or 64 bit. I suspect it's 64 but what do I know? So I pop open process monitor and see which explorer.exe is being run (there's a few locations for it on the system). Looks like by default it uses `C:\windows\explorer.exe` so let's sigcheck it out:

```>>> sigcheck.exe C:\Windows\explorer.exe

Sigcheck v2.80 - File version and signature viewer
Copyright (C) 2004-2020 Mark Russinovich
Sysinternals - www.sysinternals.com

c:\windows\explorer.exe:
        Verified:       Signed
        Signing date:   6:00 PM 10/22/2020
        Publisher:      Microsoft Windows
        Company:        Microsoft Corporation
        Description:    Windows Explorer
        Product:        Microsoft« Windows« Operating System
        Prod version:   10.0.19041.610
        File version:   10.0.19041.610 (WinBuild.160101.0800)
        MachineType:    64-bit
C:\Users\punkpc\Desktop\cmder
```
OMG I WAS RIGHT ABOUT SOMETHING. Anyway, now we know that we need to use the 64-bit DR client. So let's do that. OK done, it wasn't that exciting:

```
C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64
Î»  .\drrun.exe -t drcov -- "C:\Windows\explorer.exe" "C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64"
C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64
Î»  ls


    Directory: C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         12/7/2020   7:18 PM         207360 balloon.exe
-a----         12/7/2020   7:18 PM        4689920 balloon.pdb
-a----         12/7/2020   7:18 PM         120320 closewnd.exe
-a----         12/7/2020   7:18 PM        3829760 closewnd.pdb
-a----         12/7/2020   7:18 PM         123904 create_process.exe
-a----         12/7/2020   7:18 PM        3870720 create_process.pdb
-a----         12/7/2020   7:18 PM         236544 drconfig.exe
-a----         12/7/2020   7:18 PM        5058560 drconfig.pdb
-a----         12/7/2020   7:18 PM         246272 drconfiglib.dll
-a----         12/7/2020   7:18 PM           6920 drconfiglib.lib
-a----         12/7/2020   7:18 PM         273920 DRcontrol.exe
-a----         12/7/2020   7:18 PM        4878336 DRcontrol.pdb
-a----         12/7/2020   7:33 PM         510871 drcov.explorer.exe.19900.0000.proc.log
```

You see the log file at the bottom there gives me my code coverage. Well, while that was happening I remembered a talk by Alex Ionescu called "reversing without reversing" and I realized I fucking did it again - I jumped straight into a reversing tool and forgot to do the most basic thing - look some shit up. I Googled something like "How does Windows Explorer work MSDN" and found a pretty solid resource: https://docs.microsoft.com/en-us/windows/win32/shell/developing-with-windows-explorer.

Looks like Windows explorer has some interface definitions that are well documented. In fact it has a lot of them, they all seem worth fuzzing to be honest. But let's try to stick to the original intent. This all looks worth a read, so let's do that first and get a bit of an understanding of how we can programmatically do the stuff that explorer.exe does. In particular, how does it make the decision of what to do when I, in the command line type `explorer.exe <object>`. Let's see if that's in there somewhere.

OK so from continued reading it looks like the explorer is really just an implementation of the Shell in windows. This makes sense, I've always told people that the way to think of the shell is just like an explorer window but in text. Looking back this is obvious, so a lot of the functionality I'm looking for is actually `Shell` functionality here. Cool. Another interesting tidbit, reading Common Explorer Concepts is that stuff in Windows explorer is divided into a few different things. Specifically, files, folders, and virtual folders. Virtual folders are stuff you'd see in explorer that aren't really folders, a good example is printers, you click on a printer just as if it was a folder but it's not a folder. This brings us to the most basic unit within a folder, the SHITEMID, which simply identifies some object within an Explorer folder, it looks like this:

```
typedef struct _SHITEMID { 
    USHORT cb; 
    BYTE   abID[1]; 
} SHITEMID, * LPSHITEMID;
```

From MSDN: `The abID member is the object's identifier. The length of abID is not defined, and its value is determined by the folder that contains the object.` The details on how exactly a folder's objects are enumerated are here: https://docs.microsoft.com/en-us/windows/win32/shell/folder-info. As it turns out MSDN doesn't just call them "objects" by accident, they are implementations of that dirty bastard COM objects. Anyway, I can feel us getting closer and closer to the functionality we want (and if not some additional stuff that might be worth a fuzz) when we list objects' attributes:

```
If you have an object's fully qualified path or PIDL, SHGetFileInfo provides a simple way to retrieve information about an object that is sufficient for many purposes. SHGetFileInfo takes a fully qualified path or PIDL, and returns a variety of information about the object including:

    The object's display name
    The object's attributes
    Handles to the object's icons
    A handle to the system image list
    The path of the file containing the object's icon
```

Even not having found exactly what I want yet, already I'm interested by all of this functionality. From previous fuzzing and 0-day hunting I know that anything involving a structure that reports its own size is reasonably interesting. Why? Because oftentimes you'll find that internals of whatever will trust this value, meaning we could report a USHORT of 1 and have a name for it that is 10 bytes long, the hope would be that 1 byte is allocated and Windows attempts to shove 10 bytes into the buffer, creating overflow conditions. But that's a bit specific, let's keep going and see if we can fuzz some additional functionality along with that. I also still really want to get at that code that parses stuff and determines how explorer.exe will handle the various types of objects. 

Ah and there it is: https://docs.microsoft.com/en-us/windows/win32/shell/app-registration. As it turns out there's really not any magic happening in Explorer.exe other than looking shit up in the registry. If an executable is in the registry with something like the following:

```
HKEY_CLASSES_ROOT
   Applications
      mspaint.exe
         SupportedTypes
            .bmp
            .dib
            .rle
            .jpg
            .jpeg
            .jpe
            .jfif
            .gif
            .emf
            .wmf
            .tif
            .tiff
            .png
            .ico
```

anytime we go to open a .bmp file, mspaint is going to be used. Really, what's happening is this, say the user changes the default for .mp3 files to App2ID, the registry would undergo the following changes:

```
HKEY_CLASSES_ROOT
   .mp3
      (Default) = App1ProgID
```


```
HKEY_CLASSES_ROOT
   App1ProgID
      shell
         Verb1
```


```
HKEY_CLASSES_ROOT
   App2ProgID
      shell
         Verb2
```

And that pretty much takes care of it. There's no real magic happening there much to my dismay, and of course I should have known this going in - changing the default program that something is opened with is a fairly common operation in windows. The real vulnerabilities are just the friends we made along the way.

Just kidding. OK so it doesn't work how I thought it would, but is it still fuzzable? Fuck yeah it is, let's try something, and again I write this as stream of consciousness because I think the thought process is important - I have no idea if this is actually going to work. Why don't we take the function that reads the registry keys when I open up a file with explorer.exe, and start mutating the various registry keys. All I need is to write some functions, and take in file with registry information - these files will be mutated and the registry updated with weird, mutated information. Then explorer.exe will read it, and do its thing. At this point a couple of things are going to be helpful - let's validate that the stuff that MSDN is saying is true, and let's see if there's additional, undocumented attack surface. This shouldn't be difficult to do with procmon - another windows sysinternals tool that is pretty awesome. Let's also go ahead and continue with our coverage experiment with Binary Ninja and lighthouse. OK, let's do the procmon thing first.

Let's also introduce another simple, but extremely useful tool, called Search Everything (shortened to Everything often). It allows for very fast Windows search and can be found here: https://www.voidtools.com/. OK, I open up procmon and add a filter for explorer.exe:

![procmon registry stuff](procmon_explorer.PNG)

First thing I notice is holy shit that's a lot of registry interaction. This isn't really rare, but normally you don't have to scroll down a fuckton of times to just find something that isn't registry related. This is cool and interesting, seems like the documentation is on the right track, and I can use all of this information in my fuzzing session - perhaps I'll make a list of registry stuff that this is interacting with and fuzz it all. But maybe not, let's keep at it and see where we land. I should note, this is a stylistic thing as a hacker, I often just go into stuff completely blind and have no idea if it'll work or not. Some people like to be more deliberate and planned. That's fine, but I find that even when I fail, I end up learning a ton of stuff (like the internals of explorer). Going off on tangents is not a bad thing. Keep doing it and eventually you'll be familiar with a lot of weird Windows internal stuff that isn't even covered in any literature. OK so let's keep moving. The next thing I do is note that the explorer.exe being opened is at `C:\Windows\explorer.exe`. Great, now I know which of that 5 explorer.exe's is actually being used. To recap a bit here is my basic strategy right now: I want to fuzz explorer.exe by opening a few files and a few folders and seeing what is read in the registry. I don't want a window to pop up so I'm going to reverse engineer some functions that closely emulate what explorer.exe is doing, and then write a fuzzing harness using these functions. Now, I don't want to get my small test of explorer.exe confused with the always-open explorer.exe against a file and then a folder. Closing explorer.exe just makes it restart (side note: might be interesting to see if there's an exploitable race condition there at some point, we could check exactly HOW this restart happens by killing explorer.exe, potential replace a dll or executable and have it start up some code. I doubt this would be fruitful, but Windows does things in a funny way sometimes and it's always worth a look. We'll come back to this later if I remember). So I go ahead and copy explorer.exe to my `C:\` drive and call it explorer2.exe, that way I can filter procmon against that name instead of explorer.exe which is doing a ton of stuff by just having programs open. A lot of reverse engineering is all about **data reduction**, you get bombarded with shit with all of these tools, binary ninja gives you a massive amount of instructions, etc. Don't think that a good reverse engineer is able to understand absolutely all of it - a good reverse engineer is able to sparingly use these tools after exhausting all other simpler avenues, like what I did by simply looking up how explorer.exe works. Alex Ionescu's talk Reversing Without Reversing goes so far as to even mention that you could email Microsoft employees and kindly ask how something works. I'm not doing that here, but if it comes to it and I want to more deeply understand what is happening in some system internals it may be worth it to start making some connections at MS that may be able to help answer some questions. I'm not talking about social engineering here, I'm talking about legitimately asking someone there to help you understand the system. I have a few friends at MS, and perhaps someday I'll engage them and see if they can point me to someone willing to help. But anyway, let's keep going.

I not have a copy of explorer.exe named explorer2.exe, so I filter procmon to only give me info on images that are named explorer2.exe. OK, so blank slate! Let's try opening an exe file with explorer2.exe. Output of procmon when I do this:

![procmon explorere2.exe](explorer2.PNG)


Here you can see why Windows is an interesting attack surface. If you run something like strace on an ELF you see a decent amount of output, but it's reasonably readable. Windows on the other hand, does a fuckton of stuff that seems weirdly unnecessary. Half of it fails, some of it succeeds, and all of it can be analyzed for attack surface. I'm going to try to stay focused here, but one thing that is *always* worth doing, is opening up procmon, seeing which dlls a process is loading (sometimes it loads ones that don't even exist) and checking their permissions. One day I'll write a script that does this automatically, it shouldn't take long, I just haven't done it. DLL hijacking/replacement is a very common vulnerability, so any process that elevates its privileges should be checked to see if the dlls are world (or user) writable. If they are, you've got yourself a 0-day. Even these ridiculously simple vulns can be worth a few thousand dollars. But anyway, let's keep moving with our original intent.  

Scrolling down about halfway we start to see some familiar looking stuff. Rememebr I mentioned that explorer.exe really uses Shell functions to do its thing?

```
8:47:45.3273338 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Query Value
8:47:45.3273424 AM	explorer2.exe	22164	RegOpenKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Desired Access: Query Value
8:47:45.3273556 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3273631 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3273712 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3273790 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\Attributes	SUCCESS	Type: REG_DWORD, Length: 4, Data: 538443776
8:47:45.3273860 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3273936 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3274014 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3274090 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\CallForAttributes	NAME NOT FOUND	Length: 16
8:47:45.3274158 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3274235 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3274313 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3274393 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\RestrictedAttributes	NAME NOT FOUND	Length: 16
8:47:45.3274467 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3274547 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3274627 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3274705 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\FolderValueFlags	SUCCESS	Type: REG_DWORD, Length: 4, Data: 0
8:47:45.3274782 AM	explorer2.exe	22164	RegCloseKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	
```

There we start to see a bunch of shell operations querying the registry for stuff. Let's check it for the SupportedTypes attribute that I've been talking about.

It's not worth a screenshot because what I found: nothing. This confused me for a minute as the documentation was pretty clear "Supported types", according to my understanding, is what TELLS explorer2.exe how to open a file right?? Well as it turns out this is only somewhat correct. It looks like instead of querying this itself, it delegates the SupportedTypes check to the actual process. So it seems the workflow is something like `I type explorer2.exe firefox.exe -> firefox.exe queries to see if the file I'm trying to open is in its SupportedTypes in the registry -> if yes, cool, report back and open, if no ask the user what to open it with`. In other words it just YOLOs it out and lets the process handle it for itself. Makes sense I suppose, opening a folder isn't going to have this because it's always going to be opened with explorer.exe natively.

So now I started thinking - perhaps there's a better thing to fuzz here. If I were to find, for example a buffer overflow from reading a registry key how would I weaponize this? One way it could help would be for persistence purposes, a simple registry edit could provide persistence - but that's already do-able without a memory corruption exploit. My original intent was to be able to send a file that corrupts explorer.exe when opened. It looks like this is going to be somewhat difficult as it delegates everything to the process. So perhaps my original intent is now stupid. What do we do now? I have a saying, "when you fail, admit it early and pivot" so let's pivot. We've learned about some neat potential attack surface with windows explorer. In particular it is well documented and seems quite easy to get and set attributes of an "object" in a folder. So there's two possible scenarios I could see - one would be that we create an object that doesn't make sense, is invalid in some way, and causes some kind of corruption and/or memory disclosure via its API. Seems like a good place to fuzz. The other would be to mutate objects within a folder and list them, perhaps if we make a fucked up kind of object something bad will happen. Maybe some other fucked up shit will happen, what do I know? Seems like a decent place to start. So let's start playing around with the explorer API for fun and... yeah you get it.

OK so here's a good starting point, MSDN has been kind enough to provide us with an example program!

```
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream.h>
#include <objbase.h>

int main()
{
    IShellFolder *psfParent = NULL;
    LPITEMIDLIST pidlSystem = NULL;
    LPCITEMIDLIST pidlRelative = NULL;
    STRRET strDispName;
    TCHAR szDisplayName[MAX_PATH];
    HRESULT hr;

    hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);

    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);

    if(SUCCEEDED(hr))
    {
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));
        cout << "SHGDN_NORMAL - " <<szDisplayName << '\n';
    }

    psfParent->Release();
    CoTaskMemFree(pidlSystem);

    return 0;
}
```

Along with a description of the above program:

```
The application first uses SHGetFolderLocation to retrieve the System folder's PIDL. It then calls SHBindToParent, which returns a pointer to the parent folder's IShellFolder interface, and the System folder's PIDL relative to its parent. It then uses the parent folder's IShellFolder::GetDisplayNameOf method to retrieve the display name of the System folder. Because GetDisplayNameOf returns a STRRET structure, StrRetToBuf is used to convert the display name into a normal string. After displaying the display name, the interface pointers are released and the System PIDL freed. Note that you must not free the relative PIDL returned by SHBindToParent.
```

I like this description. Not only is it very clear what's going on in the code (meaning I can edit it to create a fuzzing harness), the various objects and structures look generally interesting. That conversion into a display name just *sounds* to me like could be a bit tricky. But who knows, maybe somewhere else in there there is a gotcha for Windows developers. Fuzzing can cause some crazy shit to happen, which is exactly what we want. Looking at the `SHGetFolderLocation` API:

```
SHSTDAPI SHGetFolderLocation(
  HWND             hwnd,
  int              csidl,
  HANDLE           hToken,
  DWORD            dwFlags,
  PIDLIST_ABSOLUTE *ppidl
);
```

Ha, ok, so one of these (`HANDLE htoken`) is a process access token. I'm not totally clear here on how the API protects itself, what if I set this to an Administrator's or System's access token using something like Mimikatz? Even metasploit's meterpreter has a get_token functionality. I could get arbitrary privileged reads for stuff that I'm not even supposed to see. Let's put a pin in that for now and come back to it so we can focus on getting a fuzzer up and running. Let's throw this code into VSCode and compile it, just for a quick sanity check. Just as a quick note that second argument to SHGetFolderLocation which is set to CSIDL_SYSTEM which means we're going to get the System folder's default display name with that code. Pretty cool, things are starting to take shape here, and I think I know how I'm going to fuzz this, but I won't spoil it just yet. Let's get that compilation done:

```
C:\Users\punkpc\Desktop\0daze
λ clang-cl.exe explorer_api_get_system_display_name.cpp ole32.lib shell32.lib shlwapi.lib
explorer_api_get_system_display_name.cpp(1,17): warning: using directive refers to implicitly-defined namespace 'std'
using namespace std;
                ^
1 warning generated.

C:\Users\punkpc\Desktop\0daze
λ ls
explorer_api_get_system_display_name.cpp  explorer_api_get_system_display_name.exe*  explorer_api_get_system_display_name.obj

C:\Users\punkpc\Desktop\0daze
λ .\explorer_api_get_system_display_name.exe
SHGDN_NORMAL - System32
```

This step took me longer than I'd like to admit. But there's a good reason for it! I'm going to give libfuzzer on Windows a try. I've used it on Linux before and it's ridiculously fast (I remember getting about 22k execs/sec on a decent laptop). It seems like this is a great option right now as everything is in memory. When using libfuzzer it's a good idea to have as much loaded into memory as possible (i.e. build a static executable). So let's get cracking on defining the functions we need to define and adding libfuzzer to this code.

First let's see what else we can do with this code, in particular we're looking for what the "interesting" variables are. So let's get cracking, edit some of the code, write some hacky shit, and then fuzz away. Actually, let's start by simply commenting our code to see what each line is doing:

```
using namespace std;
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream>
#include <objbase.h>

int fuzzMeDrZaus()
{

    //This is the main "folder" interface. It is the representation of the folder in the form a COM object interface.
    IShellFolder *psfParent = NULL; 

    //Each file/folder/virtual object is represented by a series of SHITEMIDs (SH item IDs, not shite MIDs as I read it)
    //Think of SHITEMIDs as filename or folder name and the LPITEMIDLIST as a fully qualified path or rather a list of SHITEMID structures
    LPITEMIDLIST pidlSystem = NULL;
    LPCITEMIDLIST pidlRelative = NULL;

    //this is where we're gong to shove the display name when we call GetDisplayNameOf (a human readable representation of an LPCITEMIDLIST)
    //The STRRET type is specific to the IShellFolder interface and is meant to hold its string. It's really a struct but it's fucking
    //complicated to let's just pretend it's a string.
    STRRET strDispName;
 
    //This is going to hold the actual string in the above struct, so it's an array of chars
    TCHAR szDisplayName[MAX_PATH];

    //The fucking result, for error checking we're not going to do because fuck that noise (and it slows down the fuzzer)
    HRESULT hr;


    /*
    SHSTDAPI SHGetFolderLocation(
      HWND             hwnd,
      int              csidl,
      HANDLE           hToken,
      DWORD            dwFlags,
      PIDLIST_ABSOLUTE *ppidl
    );
    */
   //This is really just being used as a vessel for that second arg for which CSIDL_SYSTEM will be specified
   //A CSIDL == Constant Special Item ID List
   //This function returns the path of a folder but as an ITEMIDLIST. So this is saying gimme an ITEMIDLIST
   //that points to the special System folder (system32). Neat.
   //The last argument is where the actual ITEMIDLIST is going to be stored, i.e. what we give a fuck about
    hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);

    /*
    SHSTDAPI SHBindToParent(
      PCIDLIST_ABSOLUTE pidl,
      REFIID            riid,
      void              **ppv,
      PCUITEMID_CHILD   *ppidlLast
    );
    */
   //This function will return an interface pointer to the parent folder of pidlSystem (the ITEMIDLIST for system)
   //In other words it takes the first arg and shoves the parent of the ItemID into psfParent
   //pidRelative us the items PIDL relative to the parent folder. In other words this is saying
   //"give me the fucking parent of the thing I give you (System folder here). Give it to me as an IShellFolder item,
   //and while you're add it why don't you gimme the relative path to the parent folder too." Note we seem to be 
   //getting the System folder (i.e. "C:\Windows\System32"), grabbing the parent Windows\ and then getting the relative
   //path from there to System32 (i.e. "System32"). Seems kinda weird but OK.
    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);


    //Assuming all the above shit checks out...
    if(SUCCEEDED(hr))
    {
        //use the IShellInterface COM object interface GetDisplayNameOf for the relative path of the parent folder to the system folder, 
        //give me a normal fucking string, and shove it in strDispName, note this is going to shove System32 into *strDispName.
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));

        //Output the display name
        cout << "SHGDN_NORMAL - " << szDisplayName << '\n';
    }


    //I..... release you.
    psfParent->Release();

    //Gimme my fucking memory back.
    CoTaskMemFree(pidlSystem);

    //Yay all is well. Or is it????? (it is)
    return 0;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {



}
```

The above gives you an idea of how folders and shit work. To be honest it's pretty insane. There's really not a good visual representation of things and you're highly
dependent on the IShellFolder methods to do absolutely anything. This sucks for you coders out there, but it's pretty great for us hackers. This is needlessly complicated
and where things are needlessly complicated there are bugs when they're implemented. I don't know what chucklehead thought this was a good idea, but ok, let's roll with
it.

You can see at the bottom I'm getting ready to use LibFuzzer against this. I just need to decide which input I'm going to fuzz. I think I'm going to fuzz the GetDisplayNameOf
function at it seems to be one of the most appropriate ones as the global state is fairly well established at that point and I'm asking it to perform a weird and (for some ungodly reason) difficult transformationp into a human readable string. I'm going to just roll with this code with a slight modification, but a few of these APIs seem fuzzable. Let's just get something running and iterate over it afterward.

Anyway I'm getting tired, I thought I should probably document how long this stuff is taking me, as it has been a couple of days. So from now on I'll just do a little signing off message and mention the date, which is 12/8/2020 at 1:15 AM.










 
 
  
  




