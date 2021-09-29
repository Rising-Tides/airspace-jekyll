---
layout: fun
title: Building a Blockchain (in Go) for Fun (Pt. 2)
date: 2021-09-28 21:11:27
author: Scott Einsidler
---
## The DHT

Where we last left of, we setup a nice little local blockchain that we can run on our local network, using a proof of work consensus for posting blocks. Not bad for one hefty blog post! But we still have a long hill to climb ahead of us. And one of the problems we encountered last time was that each node had to be given an address of another node in order to connect to the network. Wouldn't it be nice if we could just have our node connect to the network on its own?

That's where a DHT, or a *Distributed Hash Table*, comes into play. The IPFS documentation (written by the same people who created the library we are using to build the network) says this about DHTs:
>A distributed hash table (DHT) is a distributed system for mapping keys to values. In IPFS, the DHT is used as the fundamental component of the content routing system and acts like a cross between a catalog and a navigation system. It maps what the user is looking for to the peer that is storing the matching content. Think of it as a huge table that stores who has what data.

We can use this distributed table to store address information of other nodes on the network. But how does this information from the table get sent to us? Now we must look into the  Kademlia algorithm! 

## The Kademlia Algorithm
The DHT is just a data structure, but there are many ways to read, write, and store data in a DHT. After all, a DHT is stored on many nodes (ideally) and how this information is sent and the DHT is implemented is entirely up to the programmer. The Kademlia algorithm is an attempt to create a DHT and a protocol in which information is stored and retrieved on the network. I'm planning on writing another blog post on the Kademlia algorithm in the future, but I'll just stick to the very basics here. If you are curious to read more about it right now, I highly recommend reading more about it [here](https://docs.ipfs.io/concepts/dht/#kademlia) on the IPFS documentation page or the original paper [here](https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf). 

Essentially, the Kademlia algorithm stores information on each node based on the distance away each node's ID is from itself using XOR as a metric of distance. *What does that even mean?* Well, with the libp2p library and on IPFS are hash values, meaning they come from a function that is hard to invert and the output is of a fixed size. To find the distance between two nodes in the Kademlia algorithm, we use the XOR operation between the address. XOR means that for every bit (a bit is 0 or 1) in the address, we compare if they are the same or not. If they are different, we write a 1, and if they are the same, we write a zero. And example would be 
```
1110 XOR 0101 = 1011
```

This is a pretty odd thing to use as metric for distance, a node with an address that is one off by this metric could be on the other side of the world! But mathematicians have a much looser definition of what you can measure distance with. While physical distance in feet and yards is pretty nice to measure the distance between cities, what if we wanted to measure what is the shortest chain of friend-of-a-friends I have to get to Kevin Bacon (the classic six degrees of Kevin Bacon)? While this certainly is a measure of relative distance that we might want to know, it certainly wouldn't help to use yards for this measurement! But I digress...

Kademlia stores information about our node addresses by putting them into buckets relative to how far away they are using our fancy XOR metric. Whenever any node receives a message from another node, the receiving node updates the DHT with the address information from the other node. This helps any node keep an updated list of live nodes, all sorted based on a distance away in the address space. When there are no nodes on the network, often times a bootstrap node is given.

That's all I'll say on the algorithm for now because we don't interact with it too much under the hood for now. But I will cover the algorithm more in depth in a future blog post as I find this algorithm pretty intriguing! 

## Let's Get to the Code!

I found out about all this cool stuff in [this example](https://github.com/libp2p/go-libp2p/tree/master/examples/chat-with-rendezvous) in the go-libp2p library. Instead of dealing with all the flag parsing in main, the example has them placed in a separate flags.go file, and I decided to do the same. The main also is basically the same as the driver in the above example, using our own functions in node.go and chain.go where it is necessary and implementing our blockchain instead of the chat application. Let's go through it! 

```go
func main() {
	// establish blockchain
	t := time.Now()
	genesisBlock := chain.Block{}
	genesisBlock = chain.Block{Index: 0, Timestamp: t.String(), BPM: 0, Hash: chain.CalculateHash(genesisBlock), PrevHash: "", Difficulty: difficulty}

	chain.Blockchain = append(chain.Blockchain, genesisBlock)
	log.SetAllLoggers(log.LevelWarn)
	log.SetLogLevel("rendezvous", "info")
	help := flag.Bool("h", false, "Display Help")
	config, err := ParseFlags()
	if err != nil {
		panic(err)
	}

	if *help {
		fmt.Println("This program demonstrates a simple p2p chat application using libp2p")
		fmt.Println()
		fmt.Println("Usage: Run './chat in two different terminals. Let them connect to the bootstrap nodes, announce themselves and connect to the peers")
		flag.PrintDefaults()
		return
	}

	// var r io.Reader
	// r = rand.Reader

	// Generate a key pair for this host. We will use it
	// to obtain a valid host ID.
	// priv, _, err := crypto.GenerateKeyPairWithReader(crypto.RSA, 2048, r)
	// if err != nil {
	// 	panic(err)
	// }

	opts := []libp2p.Option{
		libp2p.ListenAddrStrings(fmt.Sprintf(config.ListenAddresses.String())),
		// libp2p.Identity(priv),
	}	

	// // libp2p.New constructs a new libp2p Host. Other options can be added
	// // here.
	ctx := context.Background()
	
	host, err := libp2p.New(opts...)
	if err != nil {
		panic(err)
	
	}
	node := chain.Node{Ha: host}
	logger.Info("Host created. We are:", host.ID())
	logger.Info(host.Addrs())

	// // Set a function as stream handler. This function is called when a peer
	// // initiates a connection and starts a stream with this peer.
	host.SetStreamHandler(protocol.ID(config.ProtocolID), node.HandleStream)

	// // Start a DHT, for use in peer discovery. We can't just make a new DHT
	// // client because we want each peer to maintain its own local copy of the
	// // DHT, so that the bootstrapping node of the DHT can go down without
	// // inhibiting future peer discovery.
	
	kademliaDHT, err := dht.New(ctx, host)
	if err != nil {
		panic(err)
	}

	// // Bootstrap the DHT. In the default configuration, this spawns a Background
	// // thread that will refresh the peer table every five minutes.
	logger.Debug("Bootstrapping the DHT")
	if err = kademliaDHT.Bootstrap(ctx); err != nil {
		panic(err)
	}

	// // Let's connect to the bootstrap nodes first. They will tell us about the
	// // other nodes in the network.
	var wg sync.WaitGroup
	for _, peerAddr := range config.BootstrapPeers {
		peerinfo, _ := peer.AddrInfoFromP2pAddr(peerAddr)
		wg.Add(1)
		go func() {
			defer wg.Done()
			if err := host.Connect(ctx, *peerinfo); err != nil {
				logger.Warning(err)
			} else {
				logger.Info("Connection established with bootstrap node:", *peerinfo)
			}
		}()
	}
	wg.Wait()

	// We use a rendezvous point "meet me here" to announce our location.
	// This is like telling your friends to meet you at the Eiffel Tower.
	logger.Info("Announcing ourselves...")

	routingDiscovery := discovery.NewRoutingDiscovery(kademliaDHT)
	discovery.Advertise(ctx, routingDiscovery, config.RendezvousString)
	logger.Debug("Successfully announced!")

	// Now, look for others who have announced
	// This is like your friend telling you the location to meet you.
	logger.Debug("Searching for other peers...")
	peerChan, err := routingDiscovery.FindPeers(ctx, config.RendezvousString)
	if err != nil {
		panic(err)
	}
	// testing for kademliaDHT
	for peer := range peerChan {
		if peer.ID == host.ID() {
			continue
		}
		logger.Debug("Found peer:", peer)

		logger.Debug("Connecting to:", peer)
		stream, err := host.NewStream(ctx, peer.ID, protocol.ID(config.ProtocolID))

		if err != nil {
			logger.Warning("Connection failed:", err)
			fmt.Print("Peer ID: ", kademliaDHT.PeerID())
			fmt.Printf("Peer Key: ")
			fmt.Printf("%+v\n", kademliaDHT.PeerKey())
			fmt.Printf("\n")
			fmt.Printf("Routing Table Check: ")
			// fmt.Printf("%+v\n", kademliaDHT.RoutingTable().GetPeerInfos())
			continue
		} else {
			// rw := bufio.NewReadWriter(bufio.NewReader(stream), bufio.NewWriter(stream))

			// go writeData(rw)
			// go readData(rw)
			node.HandleStream(stream)
		}

		logger.Info("Connected to:", peer)
	}

	select {}
}
```
So ```main``` looks very similar the ```main``` function in previous post at the start: we record the current time and startup the blockchain with a genesis block. Next, we set up our logging (not very important) and parse command line flags. These are different flags than the previous blog post, but the most important thing is that we only need to run our program with a single flag indicating an address to listen onto. We don't need another address to listen to for any node! 

Next, we have a condition if the help flag is entered. Next create an array that is used to set our options for our host. For our setup, we only need to supply "listening addresses", which is what we give on the command line when we run the node. Then we setup the host with our options.

Our ```Node``` struct now only contains a ```host.Host``` element inside of it, and we pass our new host in a node struct. Next, we set up a connection with ```setStreamHandler```. After, we setup the infamous DHT using Kademlia and attempt to bootstrap to a node from one of our bootstrap nodes. 

After attempting boostrapping, we annouce ourselves on the network for the other incoming peers attempting to join the network. The rest of main that resides in the for loop is dedicated to finding new peers and creating streams with incoming peers on the network. 

## Easy Peasy, Automatic Peer Discovery Squeezy 

We now have automatic node discovery on our network! I thought that would be a lot harder to implement this than it was, but having the chat application example came to be very useful. Instructions on how to work the repository and the code described above will be in the branch named "pt-2" in [this](https://github.com/seinsidler/scott-chain) repository. I'm going to go in more depth on the Kademlia algorithm and DHTs in a future blog post or two, but I'll just leave this a very rough overview of the growth of this project. Feel Free to help contribute to the project and create branches, maybe your changes will be added to the main branch! 
