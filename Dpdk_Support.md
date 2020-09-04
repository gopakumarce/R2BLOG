# R2 supports DPDK (Interfacing Rust & C)

DPDK support comes in via [this commit](https://github.com/gopakumarce/R2/commit/bb588807a957f1b631d818a25ba4e757bffc8f0d)

The direction I want to steer R2 is as a container networking element. And I was thinking of trying out memif in a container environment, and I wanted to quickly try out a memif driver. And thats when I decided to pull in dpdk because it supports memif and of course plethora of other drivers. Basically writing drivers is a very time intensive activity, and drivers is not what R2 wants to focus on as of today. So the usage of DPDK will be purely restricted to just the packet Rx/Tx calls, R2 will not use any other feature from DPDK. And at some point if R2 has enough drivers, dpdk can be taken out - considering that its an FFI and all of FFI is considered unsafe.

## Trying R2 with dpdk

Follow the same instructions as in ["tryme page"](https://r2.rs/tryme/), but before doing any of those steps, create a file named /etc/r2.cfg and put the below contents in it. This will ensure that the same example runs in dpdk mode, if you do a "top -H" after the tryme.sh completes, you will see two lcore threads (1 and 2) spinning at 100% - those are the dpdk polling EAL threads. Without this config file, the tryme.sh example will make R2 run in socket mode.

```
[dpdk]
on=true
mem=128
ncores=3
```


## FFI: Interfacing with C code

Obviously, calling DPDK APIs and using various dpdk structures etc. is possible only if there is a Rust defenition of those APIs and structures - and then we can mark them as "extern C" to indicate they are C compiled APIs - a one line summary of Rust's Foreign Function Interface aka FFI. But there is a gazillion number of dpdk apis and structures - there is too many, even if we use only a handful of them. I would have given up and not done this work at all if I had to sit down and hand code these APIs and structures.

Thankfully that was not needed. There is a utility called bindgen that uses help from the llvm compiler to take a piece of C code and generate rust equivalent apis and structures "automatically". See the [section on bindgen](https://r2.rs/bindgen/) to see the few basic steps I had to do to generate automatic bindings. But it is not without its own share of problems, as the [section on bindgen](https://r2.rs/bindgen/) shows, there are some manual steps to get rid of some nested __align__ attribute which bindgen doesnt recognize etc. And also bindgen obviously does not generate bindings for inline functions - and there is a truck load of inline utility APIs in dpdk. So those had to be hand coded, and that was not a lot of fun. But I cant complain because if not for bindgen this whole work would not even have been possible. I hear there is something better than bindgen that works on c++ code, have not checked that out. But I dont anticipate needing a lot of bindings to dpdk code because like I mentioned in the beginning, my need is just to use the dpdk pools and Rx/Tx APIs, I dont plan to use any other dpdk features.

## DPDK plugging into the R2 architecture

There was no architecture level changes needed to just get dpdk to Rx/Tx packets. Obviously there was a lot of pointer manipulation for making dpdk mbufs usable with R2, but otherwise dpdk drivers just plugin as an IFNode graph node in the R2 graph. And finally the entire R2 graph is run as a dpdk EAL thread. See [details in](https://r2.rs/dpdk/) And there can be as many threads as required with the ports split across the threads - the simple ["tryme" example](https://r2.rs/tryme/) can be run with dpdk enabled and two threads, one handling one port each. There needs to be a config file that turns on dpdk also [documented in](https://r2.rs/dpdk/)

## More work with dpdk

Some more items to handle over time

1. DPDK in this this first commit just uses AF_PACKET driver (running with --no-pci), but nothing prevents it from any driver - although a PCI driver needs more work in terms of unbinding linux drivers etc.., but thats all script work outside R2 itself. 
2. Also we havent really used hugepages (runnig with --no-huge), that again is a script work outside of R2. 
3. The DPDK threads run 100% by default, as seen when using the ["tryme" example](https://r2.rs/tryme/) with dpdk on. Some way to modulate them to not use 100% would be useful

## Next steps for R2

The next step is to investigate the CNI (Container Network Interface) mechanism in Kubernetes and to see what it takes to plugin R2 as the packet router in Kubernetes. Also a small raspberry-pi running R2 to use as a home router is also in the works. Stay tuned!