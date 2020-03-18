# Introducing R2, a Router in Rust

R2 began from a desire to learn the internals of the HFSC scheduler by writing it from scratch. It started with our decision to use HFSC at a startup I was working at. We were having troubles with a home grown scheduler and decided to stop investing time and effort on that - rightly so since our engineering team's size was in single digits. And the question was which open source scheduler to use. Someone else in the team had used BSD's HFSC before, so the decision was quick. I took on the task of porting it and making it work the way we wanted. I didn't want to just port it blindly without understanding the internals, because I knew I would have to fix bugs in it :). There really is just ONE piece of documentation on hfsc, which is the original paper with densely packed theorems and math. 

I decided I had to understand every bit of that math and every single proof in there if I have to use this scheduler. But the startup work left me with hardly any free time, and my code was already shipping before I could understand the paper. Apart from a couple of scale related issues that I had to fix (BSD version wasn't built to scale), it worked just fine. But I always had the underlying anxiety that one day there would be a bug which required deep understanding of the internals to fix. So I had a hard copy of the paper in my bag wherever I travelled, especially in long flights where I could do nothing else, I would read the paper. My constant carrying of the paper was a running joke among my colleagues!

Years flew by, and my anxieties went away, only because we stopped using HFSC and built something custom as the startup grew. But that paper remained in my bag and I kept reading it in every flight, it still stays in my bag wherever I travel. And at some point I thought I understood enough of the paper that it was time to write the code myself, but in a new language. Go was the popular kid in town and I thought I will write it in Go. At the same time my brother was experimenting with this new kid in town Rust and writing blogs about his experiments, so I started looking at Rust also. And for whatever reason, I just liked Rust more than Go and dropped Go and switched to Rust. And at some point when I got enough understanding of Rust and I was starting to write the scheduler, I thought why just write a scheduler, why not write an entire packet forwarding system. By then I had worked on plenty of networking code bases big and small, good and bad, success and failures, proprietary and open. And I had seen good designs and bad designs from others, made good designs and bad designs myself, and I thought why not put all the learnings together and build something bigger.

## About R2

I have attempted to provide very detailed documentation of R2 for anyone who is interested in looking, contributing or forking off on their own and doing whatever they want with it (hopefully contributing back). <https://r2.rs> has all the information, with clear directions on how to go about with R2. The code for R2 is at <https://github.com/gopakumarce/R2> - all the R2 documentation seen on the website is also in the git repository as markdown files. Feedbacks as comments on the blog or on github are all welcome! My hope is that an entry point that would be less 'feature hungry' would be as a vswitch in a virtual environment, maybe as a user space container networking alternative.

The open source engine https://fd.io, has been a huge inspiration for developing R2.

## My experience with Rust

I have to talk about Rust in detail here obviously because thats the language R2 is written in, and its a reasonably new language. I will not get into the "why not C" discussion, its simply absurd to be writing anything new in C these days, thats about it. Yes there are bazillions of libraries and years worth of developed stuff one can leverage with C, its probably hard to catch up blah blah as many people would argue - but by that token nothing new would ever get developed in this world! And whatever is absolutely essentially required from C, Rust has a very nice FFI interface.

The summary is that I think Rust is ready for big prime time, and of course its supported by the recent announcements of multple Amazon projects (Firecracker, Bottlerocket) using Rust and some job postings mentioining Apple's networking and storage divisions using Rust. I believe networking is the domain that is a perfect fit and use case for Rust, and hence R2!

### 'Bump in the road' - Learning curve

First of all, Rust has a tremendous eco system of documentation, books, examples, wonderfully upto date crates.io APIs, extremely responsive and helpful community etc.. I have asked many questions and got useful answers often within minutes! So Kudos to Rust community for that! Inspite of all that, learning Rust was not easy for me, theres no hiding that fact. It took me minimum two, or maybe three months before I got productive with Rust. Note that it took that long because I was trying to architect something from scratch and I really had to understand as much as I could to make the right choices. If someone is jumping into an existing project and starting the learning curve by fixing bugs and doing enhancements, I think then the curve is probably a month, and I think that would be a much better way to start off with Rust. And its probably even more easier if you are working in a team environment coming into a Rust project, where you have people to whom you can ask questions and get answers.

I have primarily been a C systems programmer all my life, and have dealt with all flavours of the usual corruptions and race conditions and multi threading issues and bugs in lock free algorithms etc.. I believe I could have got upto speed in half the time if the approach to learning Rust was different. Here is my personal opinion on how it could have been different for me. And I emphasise this is a personal take because thats how I learn, and other people might learn differently.

Rust has basically crystallized all the good systems programming practices into the language/compiler itself. AFTER I got a good grasp over Rust, I could understand for example that 'interior mutability' is basically a way to safely change contents of a shared object - simplest example being a lock or some atomic ops, in the multi threaded case. And that the concepts like Cell/Refcell are nothing but their equivalents in a single threaded case. IMO it would be easier if the documentations say 'hey, if you come from C, here is what you would do to achieve the same things'. If we search for questions about Cell/RefCell etc.., its evident that its a very confusing topic for people - and I dont think it has to be confusing! There are plenty of other cases (like Send/Sync) where what really is needed is not the theory, but examples saying 'if you do THIS in C, its probably buggy, and hence Rust tells you that you cant 'Send' this data to another thread'

The other area where people complain a lot is about lifetimes and lifetime elision etc.. - there the basic concepts are actually well explained in all documents, but IMO where it should have stopped is by saying that 'this is ALL you need to know to build most systems, dont get stuck on all the possible flavors of lifetime usages, you wont need it. Not just that, if you are getting bogged down by lifetime errors, you probably dont have the right design' ! And provide examples of where you can change the design to avoid getting bogged down with lifetime related errors - for example if one has to design a graph data structure where graph nodes only get added and never gets removed, thinking of having a vector of nodes and referring to the nodes via indices rather than the node pointers will be an alternative.

I think what Rust really needs is one good 'real world' reference application which is not too big, and is not too small. It should be a 'real' application people can use (like a gzip or something), it should cover the most important functionalities of Rust (including multi threading, Send/Sync etc.). It should avoid the peripheral cases (like no fancy IterMuts and fancy lifetime declarations). And its architecture should be very well documented, code should be well documented. Then people can start with the architecture of that application, read the code to see how that app does sharing of data across threads or implements a graph (as examples). A reference Rust app with good documentation will go a long way in easing the learning pain!

For those who are interested in R2, apart from the very detailed documentation I have provided on the architecture etc.., I also intend to write more blogs connecting the architecture to how its implemented using Rust, what are the possible alternatives etc.. - hopefully they act as a fast onramp to R2 and Rust. So watch out this space for more blogs.

### Post 'bump in the road'

So once I got a hang of Rust, the thing I quickly realized is that Rust will not tolerate 'mindless coding'. I generally am a person who likes to chalk out every single bit of the design in my head before I start writing code, so that worked very well with Rust and mostly it was frictionless. If you are the kind who writes a bunch of code and then decides to think whether its correct or not, Rust is not the language for you - but well in that case you shouldnt be coding either ;-).

But there would be those occassions where I would be like 'I just want to test if this will work, let me quickly pass a reference to this object to that thread and just see if it works and then ill think of the right approach'. Thats where Rust would be uncompromising - it WILL complain that your reference cant be sent to that thread for XYZ reason and then you have to go back and change multiple APIs to change the object to be reference counted or whatever else is thread safe. Doesnt matter whether you want to quickly test something or you are going to ship something, you just have to do it the right way. So 'doing it the right way' from start is enforced and eventually becomes an automatic habit - and I just love that! Often 'quick tests' are what becomes shipping code and I love it that even 'quick tests' cant take short cuts. And over time the brain naturally learns all the right patterns and it just becomes a habit.

The other thing I really loved with Rust is code reorgs - a HUGE reason why people are scared of touching ugly-but-working code is because they are worried if they will miss something after the reorg. Like 'what if I missed adding this case to all the case statements' or 'what if after the reorg I am now calling an API which is not thread safe' ? The former a decent editor 'might' be able to catch, the latter nothing can help you figure out. And the problem with tests (if there are any!) is that its most likely not going to catch the latter thread-safety issue, those things happen in scale and stress environments and then its already too late.

Reorganizing code is a pleasure with Rust - the case statement example is straightforward, missing case statements wont compile. The thread safety issue is also detected at compile time in case now your code is not thread safe any more post the reorg. So you can reorg the code, and fix the compile errors one by one and once it compiles, there is a very high chance you havent broken anything. If its C, if you reorg, all bets are off - compiling C code means NOTHING. Reworking and reorganizing code is the only way to keep a code base pristine - and if the language gives you peace of mind while doing that, to me that itself is a HUGE plus!

The other things people often talk about are the tooling and unit testing. And yes I too agree with that and dont have to repeat it. Cargo build system took me like just 30 minutes to learn. And writing cargo unit tests is a very pleasant experience!

So overall the Rust experience has been very satisfying. And there is a feeling of 'excitement' when programming with Rust, knowing that the language takes care of a lot of your headaches. In all fairness I have just used C as my main programming language and then Python as the secondary one, and javascript, tcl etc.. on occassions as hobby. Learning Python was sweet in the beginning, till I had to use it more and then it turned sour. With Rust I feel there is a bit of sourness in the beginning and then it just turns sweet over time.

Anyways, come back for more blogs that talk about R2 and Rust, hopefully it helps in learning R2 and/or Rust.