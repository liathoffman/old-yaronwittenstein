# The importance of unlearning


The world of software is changing consistently at a very fast pace.
Yesterday’s axioms might be tomorrow’s anti-patterns.

Newborn technologies rise to popularity and most of them will become obsolete sooner than expected and hardware advancements make game changers for things that were considered as science-fiction a few years ago.

The only certainty is that we don’t know what the future will bring us.

Here is a partial list of recent years emerging technologies:

1. **Blockchain**
Needless to say that blockchain has a good chance to disrupt the world as we know today

1. **Machine Learning Renaissance**
The increase of Hardware capabilities brought a new wave of tools and disruption to many industries that take advantage of Machine Learning

1. **Containers**
Now it’s much easier to deploy self-contained isolated components to the cloud. Containers orchestration infrastructures that only giants like *Google* had in-house are now publicly available for everyone (Kubernetes is the king here for now but the future will bring new tools for sure)

1. **WebAssembly**
The motivation is to have one universal bytecode that will be used not only on the browsers but on everything (desktop/smartphones/god knows).
This technology will transform the portability of software between platforms

1. **Rust**
One of the most innovative programming languages ever invented,
makes system programming a real joy. The most loved programming language in *Stack Overflow survey* for the last 4 years

1. **Elixir**
The solution to how to make Erlang virtual machine (called BEAM) mainstream, mainly thanks to its Ruby-like syntax and great tooling

1. **Serverless**
Why deploy containers to the cloud when we want only a single function that doesn’t need to be up 24x7?

1. **GraphQL**
REST API is not ideal for any problem, many data models are actually viewed better as a graph and this protocol plays nicely with them

1. **Redis**
The innovative key-value database having so many data-structures.
The most loved database in *Stack Overflow survey* for the last 3 years

1. **Kafka**
Distributed commit log that serves as the single point of truth of a system data. It makes it very easy to add/remove independent producers/consumers for our system architecture.
Thanks to Kafka it's easy to select a checkpoint and *“re-play”* historical data.

On the one hand, it’s an exciting time to be a software developer.
On the other hand, it means that in order to keep up, we must not stop learning or we’ll become irrelevant

I’ve always loved (and still do) learning.
I love reading technical books (I own too many).
I love reading beautiful pieces of source code.
I always seek for ways to be more productive and get out of my comfort zone.

**But, I think there is one important thing that people often overlook and is the importance of unlearning things**

Let’s review again the above list of technologies:

1. **Blockchain**
In order to understand how blockchain technologies work, we must forget about the centralized middleman model.

1. **Machine Learning**
For example, we need to accept here that computer neural networks can output results which we’ll be impossible for us to understand exactly where they came from

1. **Containers**
Containers disrupted the DevOps world

1. **WebAssembly**
Will open many opportunities, the obvious is using new programming languages for Frontend Programming. Another world of possibilities will be consuming WebAssembly libraries and using them within our applications. For example, a Rust program that will initiate a WebAssembly runtime hosting other wasm programs that were originally coded in Golang/Swift/other)

1. **Rust**
Learning the Rust programming language [requires us to think on resources differently](https://www.influxdata.com/blog/rust-can-be-difficult-to-learn-and-frustrating-but-its-also-the-most-exciting-thing-in-software-development-in-a-long-time/) in terms of borrowing/ownership.

1. **Elixir**
[The Let it crash](https://ferd.ca/the-zen-of-erlang.html) philosophy of Erlang asks us to accept unknowns and that things will break and prepare for them from the beginning

1. **Serverless**
Asks us to stop thinking only about services but also about single functions. It opens new opportunities to save money since we only pay for when the usage of a function (which might not be available 24x7)

1. **GraphQL**
Requires us to accept that REST isn’t always the best for any scenario and that sometimes there is actually a graph. (I personally think that starting from a REST API and gradually move towards a GraphQL is less risky for a green-field project)

1. **Redis**
Requires us to represent data in many more shapes: Key-Value / Hashes / Set / SortedSets / Lists etc.

1. **Kafka**
Once we start viewing our [system data as a log](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying), we can think more about replaceable components on our system. For example, we can easily add a new consumer and make it process data and spill it into a new database and know it won’t interfere with the rest of the system.

Unlearning means we need to accept that there might be more than one solution to a given problem, and this solution might be counterintuitive for us.

It also requires humility and willingness to start over in order to gain a new perspective and extend our toolbox.

Doing unlearning is a continuous inner fight. If done properly can be a huge enabler and if not, then we risk missing growth opportunities.
> **A change in perspective is worth 80 IQ points (Alan Kay)**
