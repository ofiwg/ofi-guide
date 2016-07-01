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
## Connected (TCP) Communication
## Connectionless (UDP) Communication
## Advantages
## Disadvantages

# High-Performance Networking
## Avoiding Memory Copies
### Network Buffers
### Resource Management
## Asynchronous Operations
### Interrupts and Signals
### Event Queues
## Direct Hardware Access
### Kernel Bypass
### Direct Data Placement

# Designing Interfaces for Performance
## Call Setup Costs
## Branches and Loops
## Command Formatting
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
