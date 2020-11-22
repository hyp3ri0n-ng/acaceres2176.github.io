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




