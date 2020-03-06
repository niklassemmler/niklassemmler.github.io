There are two types of system prevalent in todays data centers,
service-oriented architectures and Big Data frameworks. Both are attempts to
simplify the design of programs in a distributed setting.

In the service-oriented architecture (the extreme case being the microservice
architecture) services communicate via request-response messages over REST APIs.
Each top-level request issued to a service is answered by querying further
services, which in turn might involve even more services.  The resulting
communication pattern is a tree that fans out till all pieces of the work for
the request are done. Then the tree of requests retracts in the form of
responses till a response to the
original request can be issued.

The family of Big Data frameworks started with the original MapReduce framework
in 2004. Since then multiple additions and revisions have been implemented,
leading to, amongst others, the frameworks Flink and Spark. Although these
frameworks differ in many features, they all adhere to the Dataflow computing
paradigm. In this paradigm, the data is sent over a set of statically deployed
operators. Each operator manipulates incoming data items and then sends it to
the next operator.

Both the service-oriented architecture and the Big Data frameworks are attempts
at simplifying the development of applications that span more than a single
machine. Similarly they both depend on a common resource abstraction: Containers.
Linux containers consist of a minimal combination of memory and CPUs. Containers are
more lightweight than full-blown virtual machines as they re-use the Kernel of
the host machine and at the same time easier to use as they usually come with an existing
software stack (CoreOS).

Even though Containers have received great interest from the industry, a number
of open questions persist. The most prevalent is how to connect containers distributed
over multiple machines can be connected with networks while isolating different
applications at the same time. Similarly so far there seems to have been no
attempts made at implementing a scheme to share the bandwidth fairly of the
network (fair or depending on prioritization).

Furthermore most frameworks around Containers, as well as the two system types
mentioned above have been designed for the single data center and it is unclear
how they would behave for multiple data centers connected over wide-area
networks. 
