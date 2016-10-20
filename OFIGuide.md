---
layout: page
title: High Performance Network Programming with OFI
tagline: Libfabric Programmer's Guide
---
{% include JB/setup %}


# Introduction

OpenFabrics Interfaces, or OFI, is a framework focused on exporting fabric communication services to applications.  OFI is specifically designed to meet the performance and scalability requirements of high-performance computing (HPC) applications, such as MPI, SHMEM, PGAS, DBMS, and enterprise applications, running in a tightly coupled network environment.  The key components are OFI are: application interfaces, provider libraries, kernel services, daemons, and test applications.

Libfabric is a core component of OFI. It is the library that defines and exports the user-space API of OFI, and is typically the only software that applications deal with directly. Libfabric is agnostic to the underlying networking protocols, as well as the implementation of the networking devices. 

The goal of OFI, and libfabric specifically, is to define interfaces that enable a tight semantic map between applications and underlying fabric services. Specifically, libfabric software interfaces have been co-designed with fabric hardware providers and application developers, with a focus on the needs of HPC users.

This guide describes the libfabric architecture and interfaces.  It provides insight into the motivation for its design, and aims to instruct developers on how the features of libfabric may best be employed.

# Review of Sockets Communication

The sockets API is a widely used networking API.  This guide assumes that a reader has a working knowledge of programming to sockets.  It makes reference to socket based communications throughout, in an effort to help explain libfabric concepts and how they relate or differ from the socket API. To be clear, there is no intent to criticize the socket API.  The objective is to use sockets as a starting reference point in order to explain certain network features or limitations.  The following sections provide a high-level overview of socket semantics for reference.

## Connected (TCP) Communication

The most widely used type of socket is SOCK_STREAM.  This sort of socket usually runs over the TCP/IP protocol, and as a result is often referred to as a 'TCP' socket.  TCP sockets are connection-oriented, requiring an explicit connection setup before data transfers can occur.  A TCP socket can only transfer data to a single peer socket.

Applications using TCP sockets are typically labeled as either a client or server.  Server applications listen for connection request, and accept them when they occur.  Clients, on the other hand, initiate connections to the server.  After a connection has been established, data transfers between a client and server are similar.  The following code segments highlight the general flow for a sample client and server.  Error handling and some subtleties of the socket API are omitted for brevity.

```
/* Example server code flow to initiate listen */
struct addrinfo *ai, hints;
int listen_fd;

memset(&hints, 0, sizeof hints);
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;
getaddrinfo(NULL, "7471", &hints, &ai);

listen_fd = socket(ai->ai_family, SOCK_STREAM, 0);
bind(listen_fd, ai->ai_addr, ai->ai_addrlen);
freeaddrinfo(ai);

fcntl(listen_fd, F_SETFL, O_NONBLOCK);
listen(listen_fd, 128);
```

In this example, the server will listen for connection requests on port 7471 across all addresses in the system.  The call to getaddrinfo() is used to form the local socket address.  The node parameter is set to NULL, which result in a wildcard IP address being returned.  The port is hard-coded to 7471.  The AI_PASSIVE flag signifies that the address will be used by the listening side of the connection.

This example will work with both IPv4 and IPv6.  The getaddrinfo() call abstracts the address format away from the server, improving its portability.  Using the data returned by getaddrinfo(), the server allocates a socket of type SOCK_STREAM, and binds the socket to port 7471.

In practice, most enterprise-level applications make use of non-blocking sockets.  The fcntl() command sets the listening socket to non-blocking mode.  This will affect how the server processes connection requests (shown below).  Finally, the server starts listening for connection requests by calling listen.  Until listen is called, connection requests that arrive at the server will be rejected by the operating system.

```
/* Example client code flow to start connection */
struct addrinfo *ai, hints;
int client_fd;

memset(&hints, 0, sizeof hints);
hints.ai_socktype = SOCK_STREAM;
getaddrinfo("10.31.20.04", "7471", &hints, &ai);

client_fd = socket(ai->ai_family, SOCK_STREAM, 0);
fcntl(client_fd, F_SETFL, O_NONBLOCK);

connect(client_fd, ai->ai_addr, ai->ai_addrlen);
freeaddrinfo(ai);
```

Similar to the server, the client makes use of getaddrinfo().  Since the AI_PASSIVE flag is not specified, the given address is treated as that of the destination.  The client expects to reach the server at IP address 10.31.20.04, port 7471.  For this example the address is hard-coded into the client.  More typically, the address will be given to the client via the command line, through a configuration file, or from a service.  Often the port number will be well-known, and the client will find the server by name, with DNS (domain name service) providing the name to address resolution.  Fortunately, the getaddrinfo call can be used to convert host names into IP addresses.

Whether the client is given the server's network address directly or a name which must be translated into the network address, the mechanism used to provide this information to the client varies widely. This problem will increase significantly for applications that communicate with hundreds to millions of peer processes, often requiring a separate, dedicated application to solve.  For a typical client-server socket application, this is not an issue, so we will defer more discussion until later.

Using the getaddrinfo() results, the client opens a socket, configures it to non-blocking mode, and initiates the connection request.  At this point, the network stack has send a request to the server to establish the connection.  Because the socket has been set to non-blocking, the connect call will return immediately and not wait for the connection to be established.  As a result any attempt to send data at this point will likely fail.

```
/* Example server code flow to accept a connection */
struct pollfd fds;
int server_fd;

fds.fd = listen_fd;
fds.events = POLLIN;

poll(&fds, -1);

server_fd = accept(listen_fd, NULL, 0);
fcntl(server_fd, F_SETFL, O_NONBLOCK);
```

Applications that use non-blocking sockets use select() or poll() to receive notification of when a socket is ready to send or receive data.  In this case, the server wishes to know when the listening socket has a connection request to process.  It adds the listening socket to a poll set, then waits until a connection request arrives (i.e. POLLIN is true).  The poll() call blocks until POLLIN is set on the socket.  POLLIN indicates that the socket has data to accept.  Since this is a listening socket, the data is a connection request.  The server accepts the request by calling accept().  That returns a new socket to the server, which is ready for data transfers.

The server sets the new socket to non-blocking mode.  Non-blocking support is particularly important to applications that manage communication with multiple peers.

```
/* Example client code flow to establish a connection */
struct pollfd fds;
int err;
socklen_t len;

fds.fd = client_fd;
fds.events = POLLOUT;

poll(&fds, -1);

len = sizeof err;
getsockopt(client_fd, SOL_SOCKET, SO_ERROR, &err, &len);
```

The client is notified that its connection request has completed when its connecting socket is 'ready to send data' (i.e. POLLOUT is true).  The poll() call blocks until POLLOUT is set on the socket, indicating the connection request has completed.  Note that connection request may have completed with an error.  The client still needs to check if the connection attempt was successful.  That is not conveyed to the application by the poll() call.  The getsockopt() call is used to retrieve the result of the connection attempt.  If err in this example is set to 0, then the connection attempt succeeded.  The socket is now ready to send and receive data.

After a connection has been established, the process of sending or receiving data is the same for both the client and server.  The examples below differ only by name of the socket variable.

```
/* Example of client sending data to server */
struct pollfd fds;
size_t offset, size, ret;
char buf[4096];

fds.fd = client_fd;
fds.events = POLLOUT;

size = sizeof(buf);
for (offset = 0; offset < size; ) {
    poll(&fds, -1);
    
    ret = send(client_fd, buf + offset, size - offset, 0);
    offset += ret;
}
```

Network communication involves buffering of data at both the sender and receiver sides of the connection. TCP uses a credit based scheme to manage flow control to ensure that there is sufficient buffer space at the receive side of a connection to accept incoming data.  This flow control is hidden from the application by the socket API.  As a result, stream based sockets may not transfer all the data that the application requests to send as part of a single operation.

In this example, the client maintains an offset into the buffer that it wishes to send.  As data is accepted by the network, the offset increases.  The client then waits until the network is ready to accept more data before attempting another transfer.  The poll() operation supports this.  When the client socket is ready for data, it sets POLLOUT to true.  This indicates that send will transfer some additional amount of data.  The client issues a send() request for the remaining amount of buffer that it wishes to transfer.  If send() transfers less data than requested, the client updates the offset, waits for the network to become ready, then tries again.

```
/* Example of server receiving data from client */
struct pollfd fds;
size_t offset, size, ret;
char buf[4096];

fds.fd = server_fd;
fds.events = POLLIN;

size = sizeof(buf);
for (offset = 0; offset < size; ) {
    poll(&fds, -1);

    ret = recv(client_fd, buf + offset, size - offset, 0);
    offset += ret;
}
```

The flow for receiving data is similar to that used to send it.  Because of the streaming nature of the socket, there is no guarantee that the receiver will obtain all of the available data as part of a single call.  The server instead must wait until the socket is ready to receive data (POLLIN), before calling receive to obtain what data is available.  In this example, the server knows to expect exactly 4 KB of data from the client.

## Connection-less (UDP) Communication

<TO DO>

## Advantages

The socket API has two significant advantages.  First, it is available on a wide variety of operating systems and platforms, and works over the vast majority of available networking hardware.  It is easily the de facto networking API.  This by itself makes it appealing to use.

The second key advantage is that it is relatively easy to program to.  The importance of this should not be overlooked.  Networking APIs that offer access to higher performing features, but are difficult to program to correctly or well, often result in lower application performance.  This is not unlike coding an application in a higher-level language such as C or C++, versus assembly.  Although writing directly to assembly language offers the promise of being better performing, for the vast majority of developers, their applications will perform better if written in C or C++, and using an optimized compiler.  Applications should have a clear need for high-performance networking to select a socket API alternative.

## Disadvantages

When considering the problems with the socket API, we limit our discussion to the two most common sockets types: streaming (TCP) and datagram (UDP).

Most applications require that network data be sent reliably.  This invariably means using a connection-oriented TCP socket.  TCP sockets transfer data as a stream of bytes.  However, many applications operate on messages.  The result is that applications often insert headers that are simply used to convert to/from a byte stream.  These headers consume additional network bandwidth and processing.  The streaming nature of the interface also results in the application using loops as shown in the examples above to send and receive larger messsages.  The complexity of those loops can be significant if the application is managing sockets to hundreds or thousands of peers.

Another issue highlighted by the above examples deals with the asynchronous nature of network traffic.  When using a reliable transport, it is not enough to place an application's data onto the network.  If the network is busy, it could drop the packet, or the data could become corrupted during a transfer.  The data must be kept until it has been acknowledged by the peer, so that it can be resent if needed.  The socket API is defined such that the application owns the contents of its memory buffers after a socket call returns.

If we examine the socket send() call, once send() returns the application is free to modify its buffer.  The network implementation has a couple of options.  One option is for the send call to place the data directly onto the network.  The call must then block before returning to the user until the peer acknowledges that it received the data.  The send() can then return.  The obvious problem with this approach is that the application is blocked in the send() call until the network stack at the peer can process the data and generate an acknowledgement.  This can be a significant amount of time where the application is blocked and unable to process other work, such as responding to messages from other clients.

A better option is for the send() call to copy the application's data into an internal buffer.  The data transfer is then issued out of that buffer, which allows retrying the operation in case of a failure.  The send() call in this case is not blocked, but all data that passes through the network will result in a memory copy to a local buffer, even in the absense of any errors.

The immediate re-use of a data buffer after send() returns is an advantage of keeping the interface simple; however, it does result in negative impact on network performance.  For network or memory limited applications, an alternative API may be attractive.

The socket API is often considered in conjunction with TCP and UDP, that is, with protocols.  It is intentionally detached from the underlying network hardware implementation, including NICs, switches, and routers.  Access to available network features is therefore constrained by what the API can support.

# High-Performance Networking

By analyzing the socket API in the context of high-performance networking, we can start to see some features that are desirable for any network API.

## Avoiding Memory Copies

The socket API implementation usually results in data copies occuring at both the sender and the receiver.  This is a trade-off made between keeping the interface easy to use, versus providing reliability.  Ideally, all memory copies would be avoided when transferring data over the network.  There are techniques and APIs that can be used to avoid memory copies, but in practice, the cost of avoiding a copy can often be more than the copy itself, in particular for small transfers (measured in bytes, versus kilobytes or more).

To avoid a memory copy at the sender, we need to place the application data directly onto the network.  If we also want to avoid blocking the sending application, we need some way of communicating between the application and the network layer when the buffer is safe to re-use, in case it needs to be re-transmitted.  This leads us to crafting a network interface that behaves asynchronously.  The application will need to issue a request, then receive some sort of notification when the request has completed.

Avoiding a memory copy at the receiver is more challenging.  When data arrives from the network, it needs to land into an available memory buffer, or it will be dropped, resulting in the sender retransmitting the data.  If we use socket recv() semantics, the only way to avoid a copy at the receiver is for the recv() to be called before the send().  Recv() would then need to block until the data has arrived.  Not only does this block the receiver, it is impractical to use outside of an application with a simple request-reply protocol.

Instead, what is needed is a way for the receiving application to provide one or more buffers to the network for received data to land.  The network then needs to notify the application when data is available.  This sort of mechanism works well if the receiver does not care where in its memory space the data is located; it only needs to be able to process the incoming message.  (As an alternative, it is possible for the network layer to hand its buffer to the application, and have the application return the buffer when it is done with its processing.  While this approach can avoid memory copies, it suffers from a couple of drawbacks.  First, the network layer does not know what size of messages to expect, which can lead to inefficient memory use.  Second, many would consider this a more difficult programming model to use.  And finally, the network buffers would need to be mapped into the application process' memory space, which negatively impacts performance.)

In addition to processing messages, some applications want to receive data and locate it into a specific location in memory.  For example, a database may want to merge received data records into an existing table.  In such cases, even data that arrives from the network goes directly into an application's receive buffers, it may still need to be copied into its final location.  It would be ideal if the network supporting placing data that arrives from the network being placed into a specific memory buffer, with the buffer determined based on the contents of the data.

### Network Buffers

Based on the problems described above, we can start to see that avoiding memory copies depends on the ownership of the memory buffers used for network traffic.  With socket based transports, the network buffers are owned and managed by the networking stack.  This is usually handled by the operating system kernel.  However, this results in the data 'bouncing' between the application buffers and the network buffers.  By putting the application in control of managing the network buffers, we can avoid this overhead.  The cost for doing so is additional complexity in the application.

Note that even though we want the application to own the network buffers, we would still like to avoid the situation where the application implements a complex network protocol.  The trade-off is that the app provides the data buffers to the network stack, but the network stack continues to handle things like flow control, reliability, and segmentation and reassembly.

### Resource Management

We define resource management to mean properly allocating network resources in order to avoid overrunning data buffers or queues.  Flow control is a common aspect of resource management.  Without proper flow control, a sender can overrun a slow or busy receiver.  This can result in dropped packets, retransmissions, and increased network congestion.  Significant research and development has gone into implementation flow control algorithms.  Because of its complexity, it is not something that an application developer should need to deal with.  That said, there are some applications where flow control simply falls out of the network protocol.  For example, a request-reply protocol naturally has flow control built in.

For our purposes, we expand the definition of resource management beyond flow control.  Flow control typically only deals with available network buffering at a peer.  We also want to be concerned about having available space in outbound data transfer queues.  That is, as we issue commands to the local NIC to send data, that those commands can be queued at the NIC.  When we consider reliability, this means tracking outstanding requests until they have been acknowledged.  Resource management will need to ensure that we do not overflow that request queue.

Additionally, supporting asynchronous operations (described in detail below) will introduce potential new queues.  Those queues must not overflow as well.

## Asynchronous Operations

Arguably, the key feature of achieving high-performance is supporting asynchronous operations.  The socket API supports asynchronous transfers with its non-blocking mode.  However, because the API itself operates synchronously, the result is additional data copies.  For an API to be asynchronous, an application needs to be able to submit work, then later receive some sort of notification that the work is done.  In order to avoid extra memory copies, the application must agree not to modify its data buffers until the operation completes.

There are two main ways to notify an application that it is safe to re-use its data buffers.  One mechanism is for the network layer to invoke some sort of callback or send a signal to the application that the request is done.  Some asynchronous APIs use this mechanism.  The drawback of this approach is that the signal interrupts the application's processing.  This can negatively impact the CPU caches, plus requires interrupt processing.  Additionally, it is often difficult to develop an application that can handle processing a signal that can occur at anytime.

An alternative mechanism for supporting asynchronous operations is to write events into some sort of completion queue when an operation completes.  This provides a way to indicate to an application when a data transfer has completed, plus gives the application control over when and how to process completed requests.  For example, it can process requests in batches to improve code locality and performance.

### Interrupts and Signals

Interrupts are a natural extension to supporting asynchronous operations.  However, when dealing with an asynchronous API, they can negatively impact performnace.  Interrupts, even when directed to a kernel agent, can interfere with application processing.

If an application has an asynchronous interface with completed operations written into a completion queue, it is often sufficient for the application to simply check the queue for completions.  As long as the application has other work to perform, there is no need for it to block.  This alleviates the need for interrupt generation.  A NIC merely needs to write an entry into the completion queue and update a tail pointer to signal that a request is done.

If we follow this argument, then it can be beneficial to give the application control over when interrupts should occur and when to write events to some sort of wait object.  By having the application notify the network layer that it will wait until a completion occurs, we can better manage the number and type of interrupts that are generated.

### Event Queues

As outlined above, there are performance advantages to having an API that reports completions or provides other types of notification using an event queue.  A very simple type of event queue merely tracks completed operations.  As data is received or a send completes, an entry is written into the event queue.

## Direct Hardware Access

When discussing the network layer, most software implementations refer to kernel modules responsible for implementing the necessary transport and network protocols.  However, if we want network latencies to approach sub-microsecond speeds, then we need to remove as much software between the application and its access to the hardware as possible.  One way to do this is for the application to have direct access to the network interface controller's command queues.  Similarly, the NIC requires direct access to the application's data buffers and control structures, such as the above mentioned completion queues.

Note that when we speak about an application having direct access to network hardware, we're referring to the application process.  Naturally, an application is highly unlikely to code for a specific hardware NIC.  That work would be left to some sort of network library that specifically targetted the NIC.  The actual network layer, which implements the network transport, could be part of the network library or actually offloaded onto the NIC's hardware or firmware.

### Kernel Bypass

Kernel bypass is a feature that allows the application to avoid calling into the kernel for data transfer operations.  This is possible when it has direct access to the NIC hardware.  Complete kernel bypass is impractical because of security concerns and resource management constraints.  However, it is possible to avoid kernel calls for what are called 'fast-path' operations, such as send or receive.

For security and stability reasons, operating system kernels cannot rely on data that comes from user space applications.  As a result, even a simple kernel call often requires acquiring and releasing locks, with data verification checks.  If we can limit the affects of a poorly written or malicious application to its own process space, we can avoid the overhead that comes with kernel validation without impacting system stability.

### Direct Data Placement

Direct data placement means avoiding data copies when sending and receive data, plus placing received data into the correct memory buffer where needed.  On a broader scale, it is part of having direct hardware access, with the application and NIC communicating directly with common memory buffers and queues.

Direct data placement is often thought of when referring to RDMA - remote direct memory access.  RDMA includes reading and writing memory that belongs to a process running on a node that is across the network.  The memory access require that the NIC access the memory without involving the execution of the peer process.  RDMA relies on offloading the network transport onto the NIC in order to avoid interrupting the target process.

# Designing Interfaces for Performance

We want to design a network interface that can meet the requirements outlined above.  Moreover, we also want to take into account the performance of the interface itself.  It is often not obvious how an interface can adversely affect performance, versus it being a result of its implementation.  The following sections describe how interface choices can impact performance.  Of course, when we begin defining the actual APIs that an application will use, we will need to consider trading off raw performance with ease of use.

When considering performance goals for an API, we need to take into account the target application use cases.  For the purposes of this discussion, we want to consider applications that communicate with thousands to millions of peer processes.  Data transfers will include millions of small messages per second per peer, up to gigabytes of data being transfered.  At such extreme scales, even small optimizations are measurable, in terms of both performance and power.  If we have a million peers sending a millions messages per second, eliminating even a single instruction from the code path quickly multiplies to saving billions of instructions from execution when viewing the operation of the entire application.

We once again refer to the socket API as part of this discussion in order to illustrate how an API can affect performance.

```
/* Notable socket function prototypes */
/* "control" functions */
int socket(int domain, int type, int protocol);
int bind(int socket, const struct sockaddr *addr, socklen_t addrlen);
int listen(int socket, int backlog);
int accept(int socket, struct sockaddr *addr, socklen_t *addrlen);
int connect(int socket, const struct sockaddr *addr, socklen_t addrlen);
int shutdown(int socket, int how);
int close(int socket); 

/* "fast path" data operations - send only */
ssize_t send(int socket, const void *buf, size_t len, int flags);
ssize_t sendto(int socket, const void *buf, size_t len, int flags,
    const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t sendmsg(int socket, const struct msghdr *msg, int flags);
ssize_t write(int socket, const void *buf, size_t count);
ssize_t writev(int socket, const struct iovec *iov, int iovcnt);

/* "indirect" data operations */
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
int select(int nfds, fd_set *readfds, fd_set *writefds,
    fd_set *exceptfds, struct timeval *timeout); 
```

Examining this list, there are a couple of features to note.  First, there are multiple calls that can be used to send data, as well as multiple calls that can be used to wait for a non-blocking socket to become ready.  This will be discussed in more detail further on.  Second, the operations have been split into different groups (terminology is ours).  Control operations are those functions that an application seldom invokes during execution.  They often only occur as part of initialization.

Data operations, on the other hand, may be called hundreds to millions of times during an application's lifetime.  They deal directly or indirectly with transferring or receiving data over the network.  Data operations can be split into two groups.  Fast path calls interact with the network stack to immediately send or receive data.  In order to achieve high bandwidth and low latency, those operations need to be as fast as possible.  Non-fast path operations that still deal with data transfers are those calls, that while still frequently called by the application, are not as performance critical.  For example, the select() and poll() calls are used to block an application thread until a socket becomes ready.  Because those calls suspend the thread execution, performance is a lesser concern.  (Performance of those operations is still of a concern, but the cost of executing the operating system scheduler often swamps any but the most substantial performance gains.)

## Call Setup Costs

The amount of work that an application needs to perform before issuing a data transfer operation can affect performance, especially message rates.  Obviously, the more parameters an application must push on the stack to call a function increases its instruction count.  However, replacing stack variables with a single data structure does not help to reduce the setup costs.

Suppose that an application wishes to send a single data buffer of a given size to a peer.  If we examine the socket API, the best fit for such an operation is the write() call.  That call takes only those values which are necessary to perform the data transfer.  The send() call is a close second, and send() is a more natural function name for network communication, but send() requires one extra argument over write().  Other functions are even worse in terms of setup costs.  The sendmsg() function, for example, requires that the application format a data structure, the address of which is passed into the call.  This requires significantly more instructions from the application if done for every data transfer.

Even though all other send functions can be replaced by sendmsg(), it is useful to have multiple ways for the application to issue send requests.  Not only are the other calls easier to read and use (which lower software maintenance costs), but they can also improve performance.

## Branches and Loops

When designing an API, developers rarely consider how the API impacts the underlying implementation.  However, the selection of API parameters can require that the underlying implementation add branches or use control loops.  Consider the difference between the write() and writev() calls.  The latter passes in an array of I/O vectors.

```
/* Sample implementation for processing an array */
for (i = 0; i < iovcnt; i++) {
    ...
}
```

In order to process the iovec array, the natural software construct would be to use a loop to iterate over the entries.  Loops result in additional processing.  Typically, a loop requires initializing a loop control variable (e.g. i = 0), adds ALU operations (e.g. i++), and a comparison (e.g. i < iovcnt).  This overhead is necessary to handle an arbitrary number of iovec entries.  If the common case is that the application wants to send a single data buffer, write() is a better option.

In addition to control loops, an API can result in the implementation needing branches.  Branches can change the execution flow of a program, impacting processor pipelining techniques.  Processor branch prediction helps alleviate this issue.  However, while branch prediction can be correct nearly 100% of the time while running a micro-benchmark, such as a network bandwidth or latency test, with more realistic network traffic, the impact can become measurable.

We can easily see how an API can introduce branches into the code flow if we examine the send() call.  Send() takes an extra flags parameter over the write() call.  This allows the application to modify the behavior of send().  From the viewpoint of implementing send(), the flags parameter must be checked.  In the best case, this adds one additional check (flags are non-zero).  In the worst case, every valid flag may need a separate check, resulting in potentially dozens of checks.

Overall, the sockets API is well designed considering these performance implications.  It provides complex calls where they are needed, with simpler functions available that can avoid some of the overhead inherent in other calls.

## Command Formatting

The ultimate objective of invoking a network function is to transfer or receive data from the network.  In this section, we're dropping to the very bottom of the software stack to the component responsible for directly accessing the hardware.  This is usually referred to as the network driver, and its implementation is often tied to a specific piece of hardware, or a series of NICs by a single hardware vendor.

In order to signal a NIC that it should read a memory buffer and copy that data onto the network, the software driver usually needs to write some sort of command to the NIC.  To limit hardware complexity and cost, a NIC may only support a couple of command formats.  This differs from the software interfaces that we've been discussing, where we can have different APIs of varying complexity in order to reduce overhead.  There can be significant costs associated with formatting the command and posting it to the hardware.

With a standard NIC, the command is formatted by a kernel driver.  That driver sits at the bottom of the network stack servicing requests from multiple applications.  It must typically format each command only after a request has passed through the network stack.

With devices that are directly accessible by a single application, there are opportunities to use pre-formatted command structures.  The more of the command that can be initialized prior to the application submitting a network request, the more streamlined the process, and the better the performance.

As an example, a NIC needs to have the destination address as part of a send operation.  If an application is sending to a single peer, that information can be cached and be part of a pre-formatted network header.  This is only possible if the NIC driver knows that the destination will not change between sends.  The closer that the driver can be to the application, the greater the chance for optimization.  An optimal approach is for the driver to be part of a library that executes entirely within the application process space.

## Memory Footprint
### Addressing
### Communication Resources
### Network Buffering
#### Shared Receive Queues
#### Multi-Receive Buffers
## Optimal Hardware Allocation
### Sharing Command Queues
### Multiple Queues
## Progress Model Considerations
## Multi-Threading Synchronization
## Ordering
### Messages
### Completions
### Data

# OFI Architecture
## Framework versus Provider
## Control services
## Communication services
## Completion services
## Data transfer services
## Memory registration

# Object Model

# Communication Model
## Connected Communications
## Connectionless Communications

# Data Transfers
## Endpoints
### Shared Contexts
#### Receive Contexts
#### Transmit Contexts
### Scalable Endpoints
## Message transfers
## Tagged messages
## RMA
## Atomic operations

# Fabric Interfaces
## Using fi_getinfo
### Capabilities
### Mode Bits
### Addressing

# Fabric
## Attributes
## Accessing

# Domains
## Attributes
## Opening
## Memory Registration

# Endpoints
## Active
### Enabling
## Passive
## Scalable
## Resource Bindings
## EP Attributes
## Rx Attributes
## Tx Attributes

# Completions
## CQs
### Attributes
### Reading Completions
### Retrieving Errors
## Counters
### Checking Value
### Error Reporting

# Address Vectors
## Types
## Address Formats
## Insertion Methods
## Sharing Addressing Data with other Processes

# Wait and Poll Sets
## Blocking on events
### TryWait
### Wait
## Efficiently Checking Multiple Queues

# Putting It All Together
## MSG EP pingpong
## RDM EP pingpong
