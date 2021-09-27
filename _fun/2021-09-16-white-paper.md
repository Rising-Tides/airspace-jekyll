---
layout: fun
title: Building a Blockchain (in Go) for Fun (Pt. 1)
date: 2021-10-16 21:11:27
author: Scott Einsidler
---

A realization I had all too recently is that the only way to assess if you know something is by demonstrating that knowledge in some way. This is how I got into the work of dApps: I heard about the idea of smart contracts and read a lot about them, but I only had an abstract notion of the what they were. *Code that executes on a network of computers? Any somehow money is associated with these things?* I could accept some of these ideas conceptually, but how does that work out in practice? To me, there was only one way to find out. So I followed a medium article which made a simple sports betting site with smart contracts and I dove in head first.

I want to take this a step further: I've read about numerous blockchains, I've programmed and deployed smart contracts, but  I've never fully interacted with the code that constitutes the blockchain and the DHT itself. So while I can talk about various aspects of distributed networks and consensus protocols, I question how much I actually know about these topics without the experience of building a network of my own...

This is going to be a long and pretty informal series of posts detailing my experience building a blockchain network from scratch (basically) in Go. This is going to be a pretty daunting project with the goal of simply learning and implementing various concepts and exploring how they might work in practice. I have virtually no experience writing in Go and building a blockchain client, but if the only way to assess whether you know something or not is by demonstrating the knowledge, why not just try and do the damn thing! So this series is going to be the ultimate learning experience for everyone including myself, and because of this, the writing will probably be accessible to many crowds who have any computer science background or are eager to do some googling. However, this series is going to come across more as a journal and less as a tutorial series. There will be tangents, detours, and struggles all included. But the code will be included, and if you are following along and want to make changes, that would be awesome! Furthermore, I expect readers to understand, to some level of high-level abstraction, what a blockchain is. If you have no idea what a blockchain is but want a quick explainer and try to hop in to this series (good luck), I highly recommend watching [3Blue1Brown's video](https://www.youtube.com/watch?v=bBC-nXj3Ng4) on Bitcoin. It won't make you an expert in blockchains, but you'll get an intuition about them, and that's a pretty good use of 30 minutes. 

## So Let's Cheat A Little, Sue Me (Please Don't)
I am way better at tinkering a program than developing a new program from scratch. A great place to get starter code for almost anything is Medium. The great thing about Medium articles is that often times, they usually have a small amount of code and has some level of explanation, but most Medium articles offer incomplete solutions or oversimplify certain aspects that can be expanded upon with more time and focus. I don't say this to demean any Medium articles (to a certain degree, I'm doing the same thing but on our own site). In fact, I think that this gives us a perfect starting point to get started and to work on the fun stuff faster. 

The code we are going to start out with comes from [this](https://mycoralhealth.medium.com/code-a-simple-p2p-blockchain-in-go-46662601f417) tutorial and [this](https://mycoralhealth.medium.com/code-your-own-blockchain-mining-algorithm-in-go-82c6a71aba1f) tutorial done by coral health. The first tutorial sets up a blockchain that posts heartbeat BPMs from each miner and uses the libp2p library to connect each node together. The second tutorial builds a proof of work blockchain. I took the first tutorial and added the proof of work consensus from the second. I won't review what's in these tutorials, because the tutorials do a good enough job with that; but the aspects I am changing and added to will be expanded upon.

## Now to the Good Stuff
Alright, so what is the file structure of the project so far? What are we working with? 
```
│   .codecov.yml
│   .gitattributes
│   defaults.go
│   error_util.go
│   go.mod
│   libp2p.go
│   libp2p_test.go
│   options.go
│   options_filter.go
│   README.md
│
├───blockchain
│   │   main.go
│   │
│   └───chain
│           block.go
│           node.go
│
├───config
│       config.go
│       config_test.go
│       constructor_types.go
│       muxer.go
│       muxer_test.go
│       reflection_magic.go
│       reflection_magic_test.go
│       security.go
│       transport.go
│
└───p2p
    ├───discovery
    │   ├───mdns
    │   │       mdns.go
    │   │       mdns_test.go
    │   │
    │   └───mdns_legacy
    │           mdns.go
    │           mdns_test.go
    │
    ├───host
    │   ├───basic
    │   │       basic_host.go
    │   │       basic_host_test.go
    │   │       natmgr.go
    │   │
    │   ├───relay
    │   │       addrsplosion.go
    │   │       addrsplosion_test.go
    │   │       autorelay.go
    │   │       autorelay_test.go
    │   │       doc.go
    │   │       log.go
    │   │       relay.go
    │   │
    │   └───routed
    │           routed.go
    │
    ├───net
    │   │   README.md
    │   │
    │   ├───conngater
    │   │       conngater.go
    │   │       conngater_test.go
    │   │
    │   └───mock
    │           complement.go
    │           interface.go
    │           mock.go
    │           mock_conn.go
    │           mock_link.go
    │           mock_net.go
    │           mock_notif_test.go
    │           mock_peernet.go
    │           mock_printer.go
    │           mock_stream.go
    │           mock_test.go
    │           ratelimiter.go
    │
    ├───protocol
    │   ├───circuitv1
    │   │   ├───pb
    │   │   │       circuitv1.pb.go
    │   │   │       circuitv1.proto
    │   │   │       Makefile
    │   │   │
    │   │   └───relay
    │   │           options.go
    │   │           relay.go
    │   │
    │   ├───circuitv2
    │   │   ├───client
    │   │   │       client.go
    │   │   │       conn.go
    │   │   │       dial.go
    │   │   │       handlers.go
    │   │   │       listen.go
    │   │   │       reservation.go
    │   │   │       transport.go
    │   │   │
    │   │   ├───pb
    │   │   │       circuit.pb.go
    │   │   │       circuit.proto
    │   │   │       Makefile
    │   │   │       voucher.pb.go
    │   │   │       voucher.proto
    │   │   │
    │   │   ├───proto
    │   │   │       protocol.go
    │   │   │       voucher.go
    │   │   │       voucher_test.go
    │   │   │
    │   │   ├───relay
    │   │   │       acl.go
    │   │   │       constraints.go
    │   │   │       constraints_test.go
    │   │   │       options.go
    │   │   │       relay.go
    │   │   │       resources.go
    │   │   │
    │   │   ├───test
    │   │   │       compat_test.go
    │   │   │       e2e_test.go
    │   │   │       empty.go
    │   │   │
    │   │   └───util
    │   │           io.go
    │   │           ma.go
    │   │           pbconv.go
    │   │
    │   ├───holepunch
    │   │   │   coordination.go
    │   │   │   coordination_test.go
    │   │   │   tracer.go
    │   │   │
    │   │   └───pb
    │   │           holepunch.pb.go
    │   │           holepunch.proto
    │   │           Makefile
    │   │
    │   ├───identify
    │   │   │   id.go
    │   │   │   id_delta.go
    │   │   │   id_glass_test.go
    │   │   │   id_push.go
    │   │   │   id_test.go
    │   │   │   obsaddr.go
    │   │   │   obsaddr_glass_test.go
    │   │   │   obsaddr_test.go
    │   │   │   opts.go
    │   │   │   peer_loop.go
    │   │   │   peer_loop_test.go
    │   │   │
    │   │   └───pb
    │   │           identify.pb.go
    │   │           identify.proto
    │   │           Makefile
    │   │
    │   └───ping
    │           ping.go
    │           ping_test.go
    │
    └───test
        ├───backpressure
        │       backpressure.go
        │       backpressure_test.go
        │
        └───reconnects
                reconnect.go
                reconnect_test.go
```

Woah that's a lot of files! Luckily, most of these are libp2p packages and modules I kept locally to the repository. In the future, I'm going to try to use more imports that are outside of the repository to decrease the size of the project. The main folder that we are going to focus on and work in is the ```blockchain/```. In ```blockchain/```, there is the driver of the blockchain client, ```main.go```, and a subfolder, ```chain/```, with a ```node.go``` file and a ```block.go``` file. The original medium post had all the contents of the program in ```main.go```. While this is helpful for a tutorial, if we want to build out this program more easily, its best to seperate things out in seperate files where we can define different (almost) objects (Golang doesn't have objects exactly, but structs with methods are cool too) that we can build functionality as we go without needing to take 2 advil before we do it because the ```main()``` function is 1000 lines long!


The ```node.go``` file is responsible for handling the network related functions: 
-   MakeBasicHost: Create a host that is ready to connect to a peer
-   HandleStream: Creates a buffer stream for reading and writing
-   ReadData: Reads data from the buffer stream and accept new blocks given the chain is the longest
-   WriteData: Writes and sends updated blockchain to other peers, while attempting to add new blocks

The ```block.go``` file is responsible for handling all of the blockchain related structs:
-   A Block struct that contains all the info we need to store in a block
    -   For now, we are transmitting individual heartbeat measurements to emulate a hospital network use case
-   IsBlockValid: Checks if a new block is valid by verifying the Proof-of-Work is valid and that the block is indeed the next block in the chain. 
-   CalculateHash: Calculates a SHA256 hash for a given block
There is a little more to these functions, and if you are curious or lost, I highly recommend reading the Medium articles referenced to get a better insight as to what is going on within these functions. 

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
