---
layout: page
title: High Performance Network Programming with OFI
tagline: Libfabric Programmer's Guide
---
{% include JB/setup %}



# Introduction


OpenFabrics Interfaces, or OFI, is a framework focused on exporting fabric communication services to applications.  OFI is specifically designed to meet the performance and scalability requirements of high-performance computing (HPC) applications, such as MPI, SHMEM, PGAS, DBMS, and enterprise applications, running in a tightly coupled network environment.  The key components are OFI are: application interfaces, provider libraries, kernel services, daemons, and test applications.

Libfabric is a core component of OFI.  It is the library that defines and exports the user-space API of OFI, and is typically the only software that applications deal with directly.  Libfabric is supported on commonly available Linux based distributions.  Libfabric is agnostic to the underlying networking protocols, as well as the implementation of the networking devices.

Libfabric is architected to support process direct I/O.  Process direct I/O, historically referred to as RDMA, allows an application to access network resources without operating system interventions.  Data transfers can occur between networking hardware and application memory with minimal software overhead.  Although libfabric supports process direct I/O, it does not mandate any implementation or require operating system bypass.



# Review of Sockets Communication
## Connected (TCP) communication
## Connectionless (UDP) communication
## Advantages
## Disadvantages
# High-Performance Networking
## Avoiding memory copies
### Network buffers
### Resource management
## Asynchronous operations
		Interrupts and signals
		Event queues
## Direct hardware access
		Kernel bypass
		Direct data placement
# Designing Interfaces for Performance
## Call setup costs
## Branches and loops
## Command formatting
##  Memory footprint
		Addressing
		Communication resources
		Network Buffering
			Shared Rx queues
			Multi-receive buffers
## Optimal hardware allocation
		Shared queues
		Multiple queues
## Progress model considerations
## Multi-threading synchronization
## Ordering
		Message
		Completion
		Data
# OFI Architecture

# Framework versus Provider
OFI is divided into two separate components.  The main component is the OFI framework, which defines the interfaces that applications use.  The OFI frameworks provides some generic services; however, the bulk of the OFI implementation resides in the providers.  Providers plug into the framework and supply access to fabric hardware and services.  Providers are often associated with a specific hardware device or NIC.  Because of the structure of the OFI framework, applications access the provider implementation directly for most operations, in order to ensure the lowest possible software latencies.

# Control services
## Communication services
## Completion services
## Data transfer services
## Memory registration
# Object Model
# Communication Model
## Connected communications
## Connectionless communications
# Data Transfers
## Endpoints
		Shared Contexts
			Rx
			Tx
		Scalable Endpoints
## Message transfers
## Tagged messages
## RMA
## Atomic operations
# Fabric Interfaces
## fi_info / fi_getinfo
### Capabilities
### Mode bits
### Addressing
## Fabric
### Attributes
### Accessing
## Domains
		Attributes
		Opening
		Memory registration
## Endpoints
		Active
			Enabling
		Passive
		Scalable
		Resource Bindings
		EP Attributes
		Rx Attributes
		Tx Attributes
## Completions
		CQs
			Attributes
			Reading completions
			Retrieving errors
		Counters
			Checking value
			Error reporting
## Address Vectors
		Types
		Address formats
		Insertion methods
		Sharing with other processes
## Wait and Poll Sets
### Blocking on events
#### TryWait
#### Wait
### Efficiently checking multiple queues
# Putting It All Together
## MSG EP pingpong
## RDM EP pingpong
