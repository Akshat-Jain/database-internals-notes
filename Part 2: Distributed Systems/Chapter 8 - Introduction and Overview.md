# Chapter 8: Introduction and Overview

Concurrency is difficult even in a single machine, and it is even more difficult in a distributed system.

- In a concurrent system, processors use the shared memory to exchange the information.
- In a distributed system, each processor has its local state and the participants have to communicate by exchanging messages.

**Concurrent and Parallel**
- Concurrent: Happening simultaneously, but only one task is being executed at a time.
- Parallel: Multiple tasks are being executed simultaneously.

## Shared State in a Distributed System

Even if we introduce a "shared state"-equivalent in a distributed system, it is not going to be the same as a shared state in a single machine.

To access the shared state in a distributed system, the processes still have to go through the communication medium.

- Aspect 1: How long do we wait for a response from the other process?
  - This is defined by system's "synchony" - whether the system is asynchronous or there are any timing assumptions.
- Aspect 2: We don't know the reason for the delay.
  - It could be because the other process is busy, or it could be because the other process is dead.
  - It could be because the network is slow, or it could be because the network is partitioned.
  - This is defined by system's "failure model" - what kind of failures can occur and how they are handled.
- Aspect 3: System reliability
  - This is defined by system's "fault tolerance" - how the system behaves in the presence of faults.

## Fallacies of Distributed Computing

The fallacies of distributed computing are a set of assertions made by L Peter Deutsch and others at Sun Microsystems describing false assumptions that programmers new to distributed applications invariably make. Source: [Wikipedia](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing#:~:text=The%20fallacies%20of%20distributed%20computing,to%20distributed%20applications%20invariably%20make)

Unfortunately, there are more assumptions that people make.

**Processing**

- We cannot assume that processing is instantaneous.
- The message may join the pending queue on the remote server, and will be processed only after some time.

Backpressure:
- Backpressure refers to the resistance or force opposing the desired flow of data through software. [Source](https://www.educative.io/answers/techniques-to-exert-back-pressure-in-distributed-systems)
- It is a strategy that allows us to cope with producers that publish messages at a faster rate than what the consumers can process, by slowing down the producers.

**Clocks and Time**

We cannot assume that clocks on different remote machines are in sync.

**State Consistency**

- We have to be cautious of false assumptions and avoid oversimplification in understanding distributed systems.
- Loose Constraints in Distributed Algorithms: Some algorithms allow state divergence. In such cases, conflict resolution and read-time data repair are crucial.
- Eventually consistent distributed DBMS cannot assume that the schema is consistent across all nodes.

**Local and Remote Execution**

- Hiding complexity behind APIs for remote datasets can lead to issues.
- Using the same interface for local and remote execution may be misleading.
- Remote invocation is much costlier than local calls.
- Interleaving local and blocking remote calls can cause performance degradation.

**Need to Handle Failures**

- In a long-running system, nodes might be done for maintenance.
- Some distributed algorithms use heartbeat protocols and failure detectors to detect the state of the nodes.

**Network Partitions and Partial Failures**

Network Partition: When two or more servers cannot communicate with each other.

Network partitions can much more troublesome, since independent server-groups can proceed with execution and produce conflicting results.

**Cascading Failures**

Cascading failures are failures that start in one component and then spread to other components.

Example:
- A server is overloaded and starts responding slowly.
- The clients start timing out and retrying.
- The server gets even more overloaded and starts responding even more slowly.

## Distributed Systems Abstractions

We discuss common terms and abstractions used in distributed systems.

### Links

**Fair-loss link**

- A fair loss link is a link that may lose or repeat some packets.
- A sent message can be in the following states from senders' perspective:
  - Not yet delivered
  - Irrecoverably lost
  - Delivered (but the sender doesn't have a way to verify this)

**Message acknowledgments**

- To get more clarity on message status, we can use acknowledgments to verify that a message has been received.
- We use monotonically increasing message identifiers called "sequence numbers" to identify messages.

An example ACK protocol:
- The sender sends a message M(n), where n is the sequence number.
- The receiver sends an ACK(n) message back to the sender.

When A receives ACK(n), it knows that M(n) has been received by B.

**Message retransmits**

- Sender keeps sending the message indefinitely.
- This is highly impractical.

**Problem with retransmits**

- Retransmits are safe only if the message is idempotent.
- If the message is not idempotent, the receiver may end up processing the same message multiple times.
- For example, if the message is "debit $100 from account A", the receiver may end up debiting $100 multiple times.

But:
- Making every message idempotent is not practical.
- So we use deduplication to avoid processing the same message multiple times.

**Message order**

2 problems:
- Out of order messages
- Duplicate messages due to retransmits

To solve these problems, the receiver can keep a track of the following:
- The last message it received
- The last message it processed

If the receiver receives a message that is older than the last message it processed, it can ignore the message.

If the receiver receives a message that is out of order, it can put the message in a re-ordering buffer and process it later.

This solution gives us a link called "perfect link", and provides following guarantees:
1. Reliable delivery
2. No duplication
3. No messages are created out of thin air

**Exactly-once delivery**

Most real-world systems don't provide exactly-once delivery. They provide at-least-once delivery == sender retransmits the message until it receives an ACK.

Exactly-once-delivery is possible only if the sender and receiver are aware of each other's state. This is theoretically impossible in a distributed system (Two Generals' Problem).

## Two Generals' Problem

This is an experiment that shows that it is impossible to achieve a consensus between two parties in a network where communication is subject to delays and failures.

Challenge: Two generals, each commanding a separate army situated on opposite sides of an enemy city, need to coordinate a synchronized attack. They can only communicate by sending messages through a messenger. The messenger can be captured by the enemy, and the message can be lost.

General A: Sends a message M(n) to General B, where n is the sequence number.
General B: Sends an ACK(M(n)) back to General A.

For B to be sure that A has received ACK(M(n)), it has to wait for ACK(ACK(M(n))) from A.

For A to be sure that B has received ACK(ACK(M(n))), it has to wait for ACK(ACK(ACK(M(n)))) from B.

This is an infinite loop. No way for them to coordinate the attack, that is, to reach a consensus.

## FLP Impossibility

For a consensus protocol, there are 3 requirements:
1. Agreement: All non-faulty processes must agree on the same value.
2. Validity: The agreed value has to be proposed by some process.
3. Termination: All non-faulty processes must eventually reach the decision state.

FLP Impossibility: Algorithms in asynchronous systems cannot be based on timeouts, and there’s no way for a process to find out whether the other process has crashed or is simply running too slow. The paper shows that, given these assumptions, there exists no protocol that can guarantee consensus in a bounded time. No completely asynchronous consensus algorithm can tolerate the unannounced crash of even a single remote process.

## System Synchrony

Problem with asynchronous systems: The assumptions are very pessimistic and impractical.

If we loosen the assumptions, we can build practical synchronous systems. It would assume that processes are progressing at comparable rates, transimission delays are bounded, message delivery cannot take indefinite time.

Defining "synchony" in a distributed system is important since it has a direct impact on the design of the system, and the algorithms that can be used.

## Failure Models

We discuss 3 failure models:

1. Crash Faults: We assume that once a process has crashed, it remains in this state.
2. Omission Faults: We assume that the process skips some algorithm steps, or is not able to execute them, or the execution isn't visible to other processes, etc.
3. Arbitrary Faults: We assume that process continues to execute, but it can behave arbitrarily (contradiciting the algorithm steps). Example: A process decides on a value that was not proposed by any process.

## Handling Failures

We can mask failures by forming process groups and using redundancy. If a process fails, the other processes can take over, and the user won't notice the failure.

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.