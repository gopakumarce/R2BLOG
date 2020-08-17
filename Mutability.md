# Mutability 

Mutability is one of the common confusing topics for Rust beginners. There are in numerous number of articles explaining WHAT it is with hypothetical examples. I am trying to explain using a close-to-real-world example WHAT it is and WHY it is, so that we appreciate the practical benefits of this concept 

## Comparing with a C 'const'

```c
void log_pkt (const Packet *pkt)  {
    Packet *tmp = (Packet *)pkt;
    // Feel free to modify the packet via tmp!
}

void read_data(struct socket *sock) {
    Packet pkt = allocate_packet();

    // Read data from socket into pkt, run the pkt through the stack,
    // applying encapsulations etc..This API modifies the pkt obviously
    read_and_run_through_stack(sock, &pkt);

    // Log the packet, this is NOT expected to modify the pkt
    log_pkt(&pkt);

    // Send packet onto the wire, this is NOT expected to modify the pkt
    write_to_socket(&pkt);
}
```

Everyone who has worked in networking is familiar with the above song and dance with a packet. And everyone will be familiar with that one smart Aleck who decides to modify the pkt inside the log_pkt() API ! The person who made that decision was probably logging some value in the packet, found the value out of bounds and decided 'let me just fix it here'. 

C lets us declare log_pkt parameter as *const* *Packet, but as we all know, that is easy to override in multiple ways. So basically the pkt inside log_pkt() should NOT be "mutated", but we have no way to be sure about it, log_pkt() should only get an "immutable reference" to pkt. 

Why is this such a big deal ? Being able to build a mental 'model' about how and where data gets modified, is probably THE MOST critical piece of building a stable, safe and secure system. As a programmer, one has to be able to look at the code and easily answer the question 'will my data get modified at this point' ? In the above  example, if we ask the question "will my data be modified by log_pkt", there is no way to answer that till we dig deep inside log_pkt(), which will probably lead us to two dozen possible call chains. Anyone who has worked in systems area for a while will know how easy it is for bugs to happen because we made wrong assumptions about how/where data gets modified, and such bugs are a dime a dozen!

Now let us see the Rust version

```rust
log_pkt (pkt: &Packet) {

}
fn read_and_run_through_stack (sock: &Socket, pkt: &mut Packet) {
}
read_data (sock: &Socket) {
    let mut pkt: Packet = allocate_packet();

    read_and_run_through_stack(sock, &mut pkt);
    log_pkt(&pkt);
    write_to_socket(&pkt);
}
```

In the Rust version, note the keyword 'mut'. The 'let mut pkt' declaration says that 'ok, this packet is allocated with some contents, but those contents can be changed later'. And note the reference taken as '&mut pkt' - that is saying that whoever has this reference, can modify (mutate) the packet. And whoever has just an &pkt without the mut, cannot 'mutate' the packet. So now one might ask whether this mut stuff in Rust is just a glorified strict 'const' checking in C ? Let us look at the below example to see what else the
mut keyword buys us

## Mutability is not just a C const

```rust
log_pkt (pkt: &Packet) {

}
fn read_and_run_through_stack (sock: &Socket, pkt: &mut Packet) {
}
read_data (sock: &Socket) {
    let mut pkt: Packet = allocate_packet();
    let p = &mut pkt;
    ....
    if p.has_some_property {
        set_some_field = true;
    }
    read_and_run_through_stack(sock, &mut pkt);
    if set_some_field {
        p.some_field = 100;
    }
    log_pkt(&pkt);
    write_to_socket(&pkt);
}
```

The only difference from the previous version is the addition of a 'let p = &mut pkt' and a 'p.some_field = 100'. So 'p' is a mutable reference to pkt - exactly the same kind of reference being passed to read_and_run_through_stack(). So p has the power to change the contents of pkt. Now if you try to compile this example, it will not, rustc will throw an error that reads something like 'multiple mutable borrows not allowed'. What does that mean ? 

This is where the mut Vs non-mut is far more than just a 'const' checking in C. Its easier to understand the concept if we just lay it down in plain English - 'if I have a mutable reference (&mut ..) to a data, I am the ONLY ONE who can change that data as long as I am 'alive'. That is, if the data changes, it has to be because of some action done by me and only me'. So in the above example, the 'I' in the English statement above is from the point of view of 'p'. What 'p' is saying is that 'as long as i am around, I am the only one who should be able to change the data'. But clearly that is not correct, read_and_run_through_stack() can also change the data - so when p tries to do p.some_field = 100, at that point the data might already have been changed by read_and_run_through_stack(). So Rust is guaranteeing us that if someone has been granted an '&mut ' reference to a piece of data, while that someone is 'alive', no one else will get another '&mut reference'. There are various jargons used for this like 'unique aliasing', 'shared aliasing' etc.. - I would just throw the jargon to garbage as long as the concept is understood.

### Mutability: an enforced programming paradigm

Again, why is this such a big deal ? In the above example, before we call read_and_run_through_stack(), we checked whether the pkt satisfies some property (p.has_some_property) and based on that decided to modify some_field. Then we called read_and_run_through_stack() and after that we put into action our decision to modify some_field. But the decision
we took earlier was based on p.has_some_property, and read_and_run_through_stack() might have modified p.has_some_property! So our decision is potentially stale / invalid at this point. Now think about the mental agony of being a programmer who has to deal with this uncertainty - if we want to build a stable,safe,secure system, we better know EXACTLY what will happen to p.has_some_property inside read_and_run_through_stack(). If one decides to knowingly / conveniently ignore that, he/she is a bad programmer and is building a bad system. So with C, we are relying on 'good programmers' to do the right thing. And in the case of Rust, the compiler tells you 'no you cant do that'

But - isn't that too restrictive ? What if we make arguments like "I KNOW that read_and_run_through_stack() is NOT affecting has_some_property, so why don't you just let me do what I want" ? People do make arguments like that, unfortunately like we all know, when the system gets large, it degrades into a set of assumptions made by supposedly smart people who claim to know everything (and often does not) and those 'assumptions' are not known to anyone else and the system degrades very fast. And the rest of the software development cycle is very standard - hire more people to figure out and fix the assumptions, who makes more assumptions etc.. etc.. So yes its restrictive, and to build a durable system, it needs to have some restrictions, so get over it and learn to work with the restrictions - that is the message here! Internalize this concept that only one person can have a mutable 
reference at a time and rearrange your code, rework your logic, whatever it takes to work with that restriction. And of course it can be done, people are building stuff with Rust every day.

## Mutable and immutable references in the same scope

Now after the above discussion, a natural question is - what if read_and_run_through_stack() takes a non-mutable reference ? Will it work in that case, like the code below

```rust
log_pkt (pkt: &Packet) {

}
fn read_and_run_through_stack (sock: &Socket, pkt: &Packet) {
    if p.some_field == 10 && p.some_other_field == 20 {
        // take some action
    }
}
read_data (sock: &Socket) {
    let mut pkt: Packet = allocate_packet();
    let p = &mut pkt;
    ....
    p.some_field = 10;
    read_and_run_through_stack(sock, &pkt);
    p.some_other_field = 20;
    log_pkt(&pkt);
    write_to_socket(&pkt);
}
```

The major difference from the previous code example is that read_and_run_through_stack does NOT take '&mut Packet', instead it takes just '&Packet'. The code will still not compile with more or less the same kind of error as before. But why not ? Well, to understand that, lets think about the problem from a different angle, this time from the
perspective of read_and_run_through_stack(). So inside that API, as we can see, 'some action' is taken if two fields in the packet are 10 and 20 respectively. But the code is such that some_field is set before the call to the API and some_other_field is set after. Now, will that break the logic inside read_and_run_through_stack() ? Its hard to say and varies from context to context. But in general, this is another extremely commonly buggy coding pattern and most often leads to errors - one way to think of data is not as 'pieces' but as 'one chunk'.

Since in a C structure the fields some_field and some_other_field are two individual variables, we think that is two pieces of data. But really its not - really the 'data' here is the Packet 'as a whole' - clearly because various pieces of logic try to co-relate multiple fields of Packet to make a decision. And in the above example, its easy to see that the decision might be wrong.

### Again, its a programming paradigm

Rust again is doing the right thing here saying that 'the Packet is one single atomic piece of data, if I am an API reading any field from Packet, I better be sure that all  modifications to all the fields of Packet are complete by now'. Don't get confused by the 'atomic' term and misinterpret it as the x86 atomic ops etc.. - the 'atomic' here means
that all the fields together constitute one piece of data, and all of them have to be in a 'committed / non-changing state' when we read from the data. Again I wont repeat the same rant before - yes this can be restrictive, it can be difficult initially, but learning to code with that will be a fantastic learning and will produce solid code.

## The mutable reference rule

So from the first example, we found that we cant have two mutable references to the same object 'active' at the same time. The word 'active' is important, its just talking about the scope of a reference, we will come to that in a bit. We also saw that we cant have a immutable reference to the data while a mutable reference to it is active. Putting those two together, its evident that an object can have active at the same time [ONE mutable reference and NO other references] OR [any number of immutable references]. And this rule will  be referred to with a lot of jargon like multiple aliasing and what not, but the basic principle is this, and why the basic principle exists is explained with the above examples.

## The 'scope'

So coming back to our last example, lets write it with explicit 'scopes' - a scope can be thought of basically as something within an set of braces {}.

```rust
log_pkt (pkt: &Packet) {

}
fn read_and_run_through_stack (sock: &Socket, pkt: &Packet) {
    if p.some_field == 10 && p.some_other_field == 20 {
        // take some action
    }
}
read_data (sock: &Socket) {
    let mut pkt: Packet = allocate_packet();
    {// Scope of p start
        let p = &mut pkt;
        ....
        p.some_field = 10;
        read_and_run_through_stack(sock, &pkt);
        p.some_other_field = 20;
    } // Scope of p end
    log_pkt(&pkt);
    write_to_socket(&pkt);
}
```

So above we have put a scope for variable 'p' - we have explicitly demarcated within a block {} where p is first created and where p is last seen. And as we can see above, within that scope, p has a mutable reference and read_and_run_through_stack has an immutable reference, so that will not compile for reasons we discussed in detail before. Now let us rewrite the code to make it look like below

```rust
log_pkt (pkt: &Packet) {

}
fn read_and_run_through_stack (sock: &Socket, pkt: &Packet) {
    if p.some_field == 10 && p.some_other_field == 20 {
        // take some action
    }
}
read_data (sock: &Socket) {
    let mut pkt: Packet = allocate_packet();
    { // Scope of p start
        let p = &mut pkt;
        ....
        p.some_field = 10;
        p.some_other_field = 20;
    } // Scope of p ends

    { // Scope of the API
        read_and_run_through_stack(sock, &pkt);
    } // Scope of the API ends

    log_pkt(&pkt);
    write_to_socket(&pkt);
}
```

In this example as we can see, the scope of the mutable reference p to pkt ends BEFORE the scope of the immutable reference in read_and_run_through_stack starts, so this compiles
fine. Now whether this is logically what the programmer wanted to achieve etc.. is not answerable in a generic fashion, its dependent on the context. But the scope example shown  here gives us an understanding of what to look for when faced with compilation errors related to mutability. And often times rearranging the code to fix the scopes will solve the problem, and on other occasions it might be much harder and might need a complete rethink of the data access patterns in the code. But understanding WHY it is important to adhere to these principles will help us appreciate what looks like the 'pain' involved in putting in the design thinking.


