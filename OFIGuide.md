---
layout: page
title: High Performance Network Programming with OFI
tagline: Libfabric Programmer's Guide
---
{% include JB/setup %}


# Introduction
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
