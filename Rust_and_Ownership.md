# Rust and Ownership

As mentioned in the last blog, I plan to have a series of blogs trying to explain different facets of Rust through the lens of a networking system / R2. I try to present it using real problems we all have faced when using C, and showing how Rust deals with that. We will not worry about the general syntax of Rust, the concepts are very similar to C, like structures and enums and a main() function and all that. The best way to learn syntax is just consult an existing code base and/or google for it. The R2 code base itself will cover most of the essential syntax.

So in this post I explore the concept of 'ownership' which is central to Rust. If you are a C programmer, it's not a familiar concept. But of course if you have used other languages you would have come across it before, ie its not a concept unique to Rust. Lets start off with an example of a classic bug in a networking system - I can bet theres not a single networking person who has not debugged a problem like below - I should add 'debugged a problem like below AT LEAST ONCE EVERY YEAR' - thats the beauty of the problem. And of course the same problem can be translated to other domains also

## The buggy example in C

```c
// The caller of this API should **KNOW** that the packet is freed
// after this API returns
void process_condition_3(struct Packet *p) {
    send_packet_on_wire(p);
    drop(p); // The packet is freed!
}

// This API can return a 1 which means packet is consumed by the
// API and freed. It can return 0 which means packet is not consumed
// and can be processed further
int feature_a(struct Packet *p) {
    if (condition_1) {
        // do something
        if (condition_2) {
            // do something else
            if (condition_3) {
                process_condition_3(p);
                // BUG!!!: The person who added a call to the above API did 
                // not know or did not care to find out if the API frees the
                // packet, and hence we are not returning 1 here as we should
            }
        }
        // do something more
        return 0;
    } else {
        return 0;
    }
}

int main(void) {
    struct Packet *p = allocate_new_packet();

    int dropped = feature_a(p);
    if (dropped == 0) {
        // Invalid read, or even worse an invalid write into the pkt, corruption!
        read_write(p);
    }
}
```

The comments in the code block are self explanatory. Someone added a block of code inside feature_a() that calls process_condition_3() and just dint bother to check if the packet is freed inside that API. In this case you can ask who on earth would do such a stupid mistake, but of course you have seen 'real world process_condition_3() APIs' - where inside process_condition_3() would be potentially two dozen call chains and one has to sit and track every call chain to know if the packet might be freed in one of those chains or not. The three nested if-condition blocks are just to give a visual aid as to how even this simple example can end up being a real bug.

## Ownership / Moves in Rust

Here is where the concept of 'ownership' comes into picture. What exactly is 'p' in this case ? Pointer or not, p is the 'packet' - it is an object somewhere in memory that holds information about a packet. And again it does not matter whether the memory is on heap or stack etc.., its an object, its somewhere in memory, thats about it. Now here is when our years of C training kicks in and starts arguing 'no its not an object, its a POINTER to an object' etc.., and thats where we really have to start letting go of that training. The pointer does not really matter, all it matters is that we are passing a 'packet object' into APIs, and these APIs have the power to free the object. Without further verbiage, let me write the Rust equivalent

```rust
fn process_condition_3(mut p: Packet) {
    send_packet_on_wire(p);
    drop(p);
}

// This API can return a 1 which means packet is consumed by the
// API and freed. It can return 0 which means packet is not consumed
// and can be processed further
fn feature_a(mut p: Packet) -> isize {
    if (condition_1) {
        // do something
        if (condition_2) {
            // do something else
            if (condition_3) {
                process_condition_3(p);
            }
        }
        // do something more
        return 0;
    } else {
        return 0;
    }
}

fn main() {
    let mut p: Packet = allocate_new_packet();

    let dropped: isize = feature_a(p);
    if (dropped == 0) {
        // Invalid read, or even worse an invalid write into the pkt, corruption!
        read_write(p);
    }
}
```

The rust code above is syntactically correct, but missing structure defentions etc.., my goal is to show the concept and not really get into the exact code that compiles etc.. Also most of the syntax should be self explanatory except the 'mut' - ignore the 'mut' for now, pretend its not there. When this code is compiled, Rust will  complain that read_write(p) is trying to use an object thats not owned at that point! As far as Rust is concerned, the allocate_new_packet() created a new object - again, it DOES NOT MATTER whether the object is on stack or heap, the principles are the same no matter where the object is - at the point its created, main() 'owns' the object. Now the moment feature_a(p) is called, the 'ownership' of the object is transferred to feature_a(). main() CANNOT access or do anything with p beyond that point, plain and simple! It does not matter whether feature_a() drops the packet or queues it up somewhere, the packet p is 'moved' into feature_a() and hence is not available any longer for main() to use. There is ONE OWNER for the object at any point in time.

To recap the comparison with C - ignore the pointer aspect of C, the pointer/address-of etc.. needs to be slowly let go of - its just objects that we are dealing with, and we are passing objects around into APIs - thats what we were doing in the C example, thats what we are doing here also. Although someone might observe and say 'hey in C, if you are not passing a pointer to APIs, we are just making copies of the struct into those APIs, what about here' ? And thats a good question. So here with Rust, theres only one concept - we have objects in memory, it doesnt matter where in memory (stack/heap/blah), and when we call an API that obect is 'moved' to that API, ie that object is 'passed' to that API or its 'ownership' is transferred to that API - theres no copying business that happens like in C.

## References / Borrows in Rust

So now we are at a loss here - we set off trying to do the C example in Rust and now are we saying we cant do it, because read_write() API cant access packet p ? Lets try rewriting the example as below

```rust
fn process_condition_3(p: &mut Packet) {
    send_packet_on_wire(p);
    drop(p);
    // The object p is not dropped above, just the 'borrow' of p, 
    // ie the reference is dropped, so if we try access p below
    // using the same reference, it will fail compilation
    // access_p(p); <<== This will fail
}

// This API can return a 1 which means packet is consumed by the
// API and freed. It can return 0 which means packet is not consumed
// and can be processed further
fn feature_a(p: &mut Packet) -> isize {
    if (condition_1) {
        // do something
        if (condition_2) {
            // do something else
            if (condition_3) {
                process_condition_3(p);
            }
        }
        // do something more
        return 0;
    } else {
        return 0;
    }
}

fn main() {
    let mut p: Packet = allocate_new_packet();

    let dropped: isize = feature_a(&mut p);
    if (dropped == 0) {
        // Invalid read, or even worse an invalid write into the pkt, corruption!
        read_write(p);
    }
}
```

So as you can see the primary difference from the previous version is that feature_a() and process_conditon_3() takes an &mut Packet instead of a Packet. Again pretend the 'mut' keyword doesnt exist, it doesnt make much difference for this example. So what does this mean ? Well to put it simply, this means as follows - let main() keep the 'ownership' of packet p. And let feature_a() and process_conditon_3() 'borrow' the packet for the duration of those API calls. So the master / owner of packet p is still API main(), and feature_a() and process_conditon_3() 'borrows' packet p using 'references' - references are made by the '&' operator. And here lets again let go of some C habits - don't think of the & as a 'oh we are not passing the value but passing an address' etc.. - those C concepts don't make a lot of sense here. Here it just means that the object is 'owned' by main() and everyone else has a way to 'borrow' them for a while. Now again don't confuse 'ownership' as if the object is stored somewhere 'inside API main()' etc.. - where the object is stored has nothing to do with the 'ownership' here - the object can be on the stack of main() or it may be in the heap, but wherever it is, there is ONE OWNER for the object all the time - and in this case its main().

Now an astute reader might ask what does the drop(p) inside process_condition_3() do ? Well process_condition_3 has just 'borrowed' the object and hence it cant drop/free the object, so the drop(p) there would just end the 'borrow' and as the comment says, further use of that borrow will result in compilation failure. So now whatever we wanted will work fine - the call chains all go through and finally after the read_write(p) is complete, the Packet p is 'dropped/freed' - but well who does the drop/free, theres no code for that in the example! So Rust 'automatically' frees the object after read_write(p) because rust knows that there is no more 'owner' of p after that point, and hence the compiler will insert code to free the object automatically. Note that p is not garbage collected - the compiler tracks all the ownerships and adds drop/free at COMPILE TIME!

## Conclusion

So, if we notice the pattern, what exactly happened here ? What happened here is that Rust kind of 'forced' us to ensure that the object is freed after all those who are using it are done with it. And if we think about it, someone who attempts a rewrite of the spaghetti code in the very first C example would try to do EXACTLY THE SAME - the right thing to do there would be to ensure that the packet is dropped inside main() after all API calls that will possibly use the packet, are done with. Also note that later down the lane, if some huge code re-org happens and a new API comes in called process_condition_4(), which actually wants to 'own' the packet and not 'borrow' - ie its signature reads as below

```rust
fn process_condition_4 (mut p: Packet) {

}
```

And if someone tries to call process_condition_4() from feature_a(), the compilation will fail - because feature_a() does not 'own' packet p, it only has a 'borrow' and hence it cant really satisfy process_condition_4() which is asking for total 'ownership'. So then the person doing the code re-org will have to modify the process_condition_4() signature to take a reference, ie borrow the packet p - and that will ensure that process_condition_4() cannot "drop" packet p. And viola the code stays sane - the packet continues to be freed in main() once all API calls are done. Now imagine the same re-org situation with C ! Hopefully this example gives an idea of what the Rust ownership model is and how it 'forces' us to write code that is guaranteed to be semantically right. 
