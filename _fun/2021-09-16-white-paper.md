---
layout: fun
title: Building a Blockchain (in Go) for Fun (Pt. 1)
date: 2021-10-16 21:11:27
author: Scott Einsidler
---

A realization I had all too recently is that the only way to assess if you know something is by demonstrating that knowledge in some way. This is how I got into the work of dApps: I heard about the idea of smart contracts and read a lot about them, but I only had an abstract notion of the what they were. *Code that executes on a network of computers? Any somehow money is associated with these things?* I could accept some of these ideas conceptually, but how does that work out in practice? To me, there was only one way to find out. So I followed a medium article which made a simple sports betting site with smart contracts and I dove in head first.

I want to take this a step further: I've read about numerous blockchains, I've programmed and deployed smart contracts, but  I've never fully interacted with the code that constitutes the blockchain and the DHT itself. So while I can talk about various aspects of distributed networks and consensus protocols, I question how much I actually know about these topics without the experience of building a network of my own...

This is going to be a long and pretty informal series of posts detailing my experience building a blockchain network from scratch (basically) in Go. This is going to be a pretty daunting project with the goal of simply learning and implementing various concepts and exploring how they might work in practice. I have virtually no experience writing in Go and building a blockchain client, but if the only way to assess whether you know something or not is by demonstrating the knowledge, why not just try and do the damn thing! So this series is going to be the ultimate learning experience for everyone including myself, and because of this, the writing will probably be accessible to many crowds who have any computer science background or are eager to do some googling. However, this series is going to come across more as a journal and less as a tutorial series. There will be tangents, detours, and struggles all included. But the code will be included, and if you are following along and want to make changes, that would be awesome! Furthermore, I expect readers to understand, to some level of high-level abstraction, what a blockchain is. If you have no idea what a blockchain is but want a quick explainer and try to hop in to this series (good luck), I highly recommend watching [3Blue1Brown's video](https://www.youtube.com/watch?v=bBC-nXj3Ng4) on Bitcoin. It won't make you an expert in blockchains, but you'll get an intuition about them, and that's a pretty good use of 30 minutes. 

## Basic Setup
I'm going to leave the setup for each tutorial within the READMEs of each iteration of this. Branches on the Github repository will be made for each part of the series and the READMEs will be updated with the way to build and run the program we are making, along with any other pertinent information for the setup. I will say that to get the repository for yourself, run ```git clone https://github.com/seinsidler/scott-chain.git```. 

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

## Let's Look at Main!
Here is what's going on at the heart of the program and see what the program is actually doing. The obvious place to look is the ```main``` function:
```go
func main() {
	
	t := time.Now()
	genesisBlock := chain.Block{}
	genesisBlock = chain.Block{Index: 0, Timestamp: t.String(), BPM: 0, Hash: chain.CalculateHash(genesisBlock), PrevHash: "", Difficulty: difficulty, Nonce: ""}

	chain.Blockchain = append(chain.Blockchain, genesisBlock)

	// LibP2P code uses golog to log messages. They log with different
	// string IDs (i.e. "swarm"). We can control the verbosity level for
	// all loggers with:
	// golog.SetAllLoggers("DEBUG") // Change to DEBUG for extra info

	// Parse options from the command line
	listenF := flag.Int("l", 0, "wait for incoming connections")
	target := flag.String("d", "", "target peer to dial")
	secio := flag.Bool("secio", false, "enable secio")
	seed := flag.Int64("seed", 0, "set random seed for id generation")
	flag.Parse()

	if *listenF == 0 {
		log.Fatal("Please provide a port to bind on with -l")
	}

	// Make a host that listens on the given multiaddress
	ha, err := chain.MakeBasicHost(*listenF, *secio, *seed)
	if err != nil {
		log.Fatal(err)
	}

	if *target == "" {
		log.Println("listening for connections")
		// Set a stream handler on host A. /p2p/1.0.0 is
		// a user-defined protocol name.
		ha.SetStreamHandler("/p2p/1.0.0", chain.HandleStream)

		select {} // hang forever
		/**** This is where the listener code ends ****/
	} else {
		ha.SetStreamHandler("/p2p/1.0.0", chain.HandleStream)

		// The following code extracts target's peer ID from the
		// given multiaddress
		ipfsaddr, err := ma.NewMultiaddr(*target)
		if err != nil {
			log.Fatalln(err)
		}

		pid, err := ipfsaddr.ValueForProtocol(ma.P_IPFS)
		if err != nil {
			log.Fatalln(err)
		}

		peerid, err := peer.Decode(pid)
		if err != nil {
			log.Fatalln(err)
		}

		// Decapsulate the /ipfs/<peerID> part from the target
		// /ip4/<a.b.c.d>/ipfs/<peer> becomes /ip4/<a.b.c.d>
		targetPeerAddr, _ := ma.NewMultiaddr(
			fmt.Sprintf("/ipfs/%s", peer.Encode(peerid)))
		targetAddr := ipfsaddr.Decapsulate(targetPeerAddr)

		// We have a peer ID and a targetAddr so we add it to the peerstore
		// so LibP2P knows how to contact it
		ha.Peerstore().AddAddr(peerid, targetAddr, pstore.PermanentAddrTTL)

		log.Println("opening stream")
		// make a new stream from host B to host A
		// it should be handled on host A by the handler we set above because
		// we use the same /p2p/1.0.0 protocol
		s, err := ha.NewStream(context.Background(), peerid, "/p2p/1.0.0")
		if err != nil {
			log.Fatalln(err)
		}
		// Create a buffered stream so that read and writes are non blocking.
		chain.HandleStream(s)

		select {} // hang forever

	}
}
```
There is a fair bit going on here, so let's try to break this down. We start off by setting the current time to variable and creating a first block, the genesis block, and appending it to the blockchain. We need a starting block so that there is a hash that the next blocks can be created off of. 

Next, we parse through command line flag arguments. For those unfamiliar, command line flags are used so that the user can change certain variables of the program at run time or so that the user can execute a specific command. In our case, we have four potential command line arguments: listening port, target address, enabling secure transport, and a random seed for the id generation. After these are parsed, we immediately check if there was a flag entered for the listening port. We can't create the host without knowing the port.

Next, we make a basic host using the ```MakeBasicHost``` function described early. Then, the meat of the logic of the program kicks in. The rest of main is an If/Else statement that is conditional on whether or not a target address was provided. We only hit the ```if``` on the first node that is created; after the first node is created, we will print out a target address for the next node to connect to (don't worry, we are changing this feature next post). If we hit the ```if``` statement, we start set up a stream from our host and wait for a connection. 

If we hit the ```else``` statement, we still set up the stream the same way with the ```SetStreamHandler``` function, but we also parse the target address to get the address, protocol, and peerid. Then we change the peer address to take out the IPFS peer id part of the address and then we add the peer id and new target address to the peerstore. The peerstore is basically a list of our friendly peers and there information so that we can easily connect to the network if we disconnect at a later time. 

After, we create a new stream from the current host to the previous host. We then make the stream a buffered stream, so that we can read and write immediately and receive the data in chunks. ```HandleStream``` also includes the reading and writing to the blockchain. Secretly, most of the magic happens in this function. 

## What's Actually Going On?

So I described a lot about what's happening in the program at a high-level, but what's really going on? What is this piece of software anyways? 

What we have so far is a program that sets up a connection on a local network and tries to find other machines (lets call them nodes) on the network who are trying to communicate over the same protocol. Once a connection between two nodes are built, each node then can enter data (in our case at the moment, we are sending integers representing a heartbeat of a patient in BPM) and attempt to submit there data in a block to the blockchain. A block is only excepted if the node shows a proof of work and is on the longest chain. When a block is accepted and follows those conditions, it is added to the blockchain, which is updated on every node in the network. 

## What does this mean?

So when we talk about a blockchain, we really mean a consensus held by a network of nodes! A lot of the time, the focus when talking about blockchains is on the distributed ledger. But in reality, the software that goes into building a blockchain is not focused on the ledger; we can transmit and store data in many ways, theoretically, on a distributed network. The actual core of the software is setting up a network for this data to be communicated and propagated.

## What's Next for Our Blockchain?

We have a few problems that we are going to have to solve in order to make our blockchain functional:
- Persistence: Right now, when a node shuts down, the whole network shuts down and the whole blockchain is lost :(
- Automatic Node Discovery: In our current setup, we have to give our host an address to connect to. In practice, we need nodes to be able to make themselves discoverable without needing to supply an address to a live node
- Represent the Blockchain as a trie: this will help us many things. First, if we want to eventually build smart contracts and maybe integrate a vm with our blockchain, having a trie representation will help us be able to keep track of a "state" and make it possible for different functions to be called and recorded within the blockchain. Second, this will give use a way to record the whole blockchain and give nodes a way to replay the blockchain for syncing purposes. 
- Wallets and Other Addresses: Our blockchain is not made to handle transactions. We need to build this infrastructure in somehow.

Some of these, I don't yet know how to solve fully if I'm being honest, but that only makes it all the more exciting. For the next part of the series, we will be tackling the first two problems by implementing a DHT for automatic peer discovery and this will be our first introduction into a data structure and algorithm I've never heard of! Should be fun!

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
