---
layout: fun
title: Building a Blockchain (in Go) for Fun (Pt. 1)
date: 2021-10-16 21:11:27
author: Scott Einsidler
---

A realization I had all too recently is that the only way to assess if you know something is by demonstrating that knowledge in some way. This is how I got into the work of dApps: I heard about the idea of smart contracts and read a lot about them, but I only had an abstract notion of the what they were. *Code that executes on a network of computers? Any somehow money is associated with these things?* I could accept some of these ideas conceptually, but how does that work out in practice? To me, there was only one way to find out. So I followed a medium article which made a simple sports betting site with smart contracts and I dove in head first.

I want to take this a step further: I've read about numerous blockchains, I've programmed and deployed smart contracts, but  I've never fully interacted with the code that constitutes the blockchain and the DHT itself. So while I can talk about various aspects of distributed networks and consensus protocols, I question how much I actually know about these topics without the experience of building a network of my own...

This is going to be a long and pretty informal series of posts detailing my experience building a blockchain network from scratch (basically) in Go. This is going to be a pretty daunting project with the goal of simply learning and implementing various concepts and exploring how they might work in practice. I have virtually no experience writing in Go and building a blockchain client, but if the only way to assess whether you know something or not is by demonstrating the knowledge, why not just try and do the damn thing! So this series is going to be the ultimate learning experience for everyone including myself, and because of this, the writing will probably be accessible to many crowds who have any computer science background or are eager to do some googling. However, this series is going to come across more as a journal and less as a tutorial series. There will be tangents, detours, and struggles all included. But the code will be included, and if you are following along and want to make changes, that would be awesome!

## So Let's Cheat A Little, Sue Me (Please Don't)
I am way better at tinkering a program than developing a new program from scratch. A great place to get starter code for almost anything is Medium. The great thing about Medium articles is that often times, they usually have a small amount of code and has some level of explanation, but most Medium articles offer incomplete solutions or oversimplify certain aspects that can be expanded upon with more time and focus. I don't say this to demean any Medium articles (to a certain degree, I'm doing the same thing but on our own site). In fact, I think that this gives us a perfect starting point to get started and to work on the fun stuff faster. 

The code we are going to start out with comes from [this](https://mycoralhealth.medium.com/code-a-simple-p2p-blockchain-in-go-46662601f417) tutorial and [this](https://mycoralhealth.medium.com/code-your-own-blockchain-mining-algorithm-in-go-82c6a71aba1f) tutorial done by coral health. The first tutorial sets up a blockchain that posts heartbeat BPMs from each miner and uses the libp2p library to connect each node together. The second tutorial builds a proof of work blockchain. I took the first tutorial and added the proof of work consensus from the second. I won't review what's in these tutorials, because the tutorials do a good enough job with that; but the aspects I am changing and added to will be expanded upon.



[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
