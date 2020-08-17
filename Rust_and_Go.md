# Rust and Go

At the time that I had put https://r2.rs on github and written about R2 and Rust, the one question almost everyone had is why not use Go ? And at that time, beyond some cursory reading about the high level concepts in Go, I had not written a line of code in it and hence obviously could not provide any opinions on that. Finally I got time to play around with Go the last couple of weeks and to write some code and provide some (unsolicited) opinions on the comparison

## Learning Curve

First of all, I was amazed at the speed at which I could get started - I was trying to write a web server backend handling some REST calls, using the popular gorrilamux/negroni framework with mongoDB as the storage. And it took me just a day or two to get to a point where I could write code (that would compile and work) without having to google constantly. A lot of the basic syntax is close to C, and syntax basic data structures like a maps are similar to any other language and so that dint take any time. 

And the other (pleasant) surprise was that whatever shape or form one writes code in, the compiler usually doesnt complain. Want to return from a function a variable declared as what 'looks' like a stack variable ? No problem!, the compiler would internally use heap for that. And of course no need to worry about freeing stuff, the GC takes care of it. And go routines and channels are very easy to code.

Compare this with Rust which takes at least a good three months before you can stop using google to write code, I had mentioned those experiences here - https://r2.rs/introducing-r2. 

So this a huge plus for golang!

## Syntax, Features

Syntax wise go is indeed far more compact and far less verbose than Rust. Feature wise both languages are rather minimalistic in nature. Rust probably has a few more features like generics, but thats already being debated hotly in the go community and might be introduced in go also. But otherwise both are pretty boring in terms of features, and boring is good. 

So both fares more or less similarly in this matter, but I would give an edge to golang here.

## Threads Vs goroutines

I would not really consider this as a language comparison - well if someone really wants to create a pthread from go they might use a C FFI and do it, if someone wants to use something like a goroutine in Rust, a lib can always be built in Rust (maybe there is one, I havent checked). So this is not IMO a hard-wired / laid-in-stone problem in either language, so its not of much consequence.

## Memory management

Here is where things start getting less black and white

Of course go has a run time garbage collection Vs Rust that has a compile time mechanism. And since the GC makes sure that memory will not vanish as long as someone is using it, we avoid all the terrible nightmares that the Rust borrow-checker can present one with. GC might be just fine if I am developing a web backend and my primary job is handling short lived REST calls, and I have "stuff" stored in databases that I read/write off. But as a systems programmer, I am usually calculating (often unsuccesfully) my memory usage patterns and trying control it precisely with object pools and such, and in that scenario I really need full control over my memory.

So again here, it depends on what your use case is, if I am coding up a systems project, I would want to use Rust. If I am coding up a web backend, hell yes I will use go. So the natural side effect is that if I want to control memory myself and also get help from compiler to know that I wont end up with a dangling pointer, then I have to deal with stuff like the Rust borrow-checker - but as I had explained in my previous articles, if you are fighting too much with the borrow-checker, its almost always a sign that your design is messy and needs simplification.

## Sharing and protecting Memory 

And here is where things get really black and white!

So the major concepts Rust introduced are Ownership and Mutability of memory, explained in https://r2.rs/rust-and-mutability and in https://r2.rs/ownership-in-rust .. To try and summarize those discussions in a few lines here just for the sake of comparison: Rust DEMANDS that the coder be aware of precisely who is allowed to read and write from a piece of memory. For example in the multi threading scenario, if you have a variable foo (say integer) and you are passing around &foo across threads (or goroutines), golang compiler will not complain, Rust will scream like in hell.

So here I really have to give Rust a huge win. Rust *forces* coder to be aware of the characteristics of their data very precisely - at what point is it readable Vs writable, who all can read/write the data etc. And in this area, go is no different from C. Go has the recommendation that "dont communicate by sharing memory, share by communicating", thats indeed the right thing to do in many occassions but of course like in C it purely depends on the programmer having voluntarily put in sufficient design thoughts. golang provides tools to instrument code and catch some of these errors, but its still stops at a tool level.

And here, I LOVE IT that Rust tells me a lot of things I would have missed, and forces me to correct it and thus leaves me with no option but to write better code. Yes again it takes going through three months of pain to enjoy those benefits, but for me personally, its worth it. And in this aspect, golang loses out - with golang we are reduced to relying far more heavily on the programmer having a good intent.

## Summary

The summary in my mind is as follows

Are you building a product which will evolve to huge code base, where you want to control your memory allocations, and you have scenarios with memory being shared and passed around, and you want as much help as you can get from the compiler to build safe/reliable software ? Use Rust. If your code base does not have such memory needs/memory usage patterns, use Go.

Its easier said than done. A completely non-technical question has far more impact on the choice 

Are you a startup thats hoping to hire half a dozen people unfamiliar with Rust and under time crunch to build a product in under a year - well dont think too much, whatever is your memory blah, use Go. But if you are a startup with team of half a dozen people who already know Rust, then go for Rust - it will be much easier for the people who come later on to look at an existing code base and learn on the fly than build on from scratch in Rust.

Are you a huge company with gazillion people planning to build massive product over the next three years and you have all the memory characteristics explained above ? Well shame on you ;-) if you dont find a way to build a starter team with enough people knowing Rust / investing the time needed to learn it and make the right call since you have all the time and resources in the world!

