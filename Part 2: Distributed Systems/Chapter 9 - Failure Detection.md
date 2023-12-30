# Chapter 9: Failure Detection

Process related terms:
- dead/failed/crashed: process that has stopped executing its steps completely
- unresponsive/faulty/slow: suspected processes, which may actually be dead

Failure Detector:
- A local subsystem
- Responsible for identifying failed or unreachable processes
- This is done to exclude them from the algorithm
- This guarantees liveness while preserving safety

More terms related to algorithm properties:
- Liveness == guarantees that a specific intended event must occur
- Safety == guarantees that unintended events will not occur
- Completeness == every nonfaulty member should eventually notice the process failure, and the algorithm should eventually reach its final result.

Terms related to algorithm efficiency:
- Accuracy == whether or not the process failure was precisely detected
- Efficiency == how fast the failure detector can identify process failures

We cannot build a failure detector that is both accurate and efficient. These are parameters to tune according to the requirements of the system.

## Heartbeats and Pings

Many distributed systems implement failure detectors by using heartbeats.

Basic idea is to find out the state of the remote processes periodically. 2 ways to do this:
- Ping: Send a message to the remote process, and wait for a response. If the response is not received within a specified time period, the process is considered dead.
- Heartbeat: The remote process actively notifies its peers that it’s still running by sending messages to them.

Downsides:
- Precision relies on the careful selection of ping frequency and timeout values.
- It does not capture process visibility from the perspective of other processes.

## Timeout-Free Failure Detector

State/Components involved:
- Each process maintains a list of neighbors and counters associated with them.
- Processes start by sending heartbeat messages to their neighbors.
- Each message contains the path traveled by the heartbeat so far.

On receiving a heartbeat message, the process:
- increments counters for all participants present in the path.
- appends itself to the path
- sends the heartbeat to the processes that are not present there.

This approach allows us to correctly mark an unreachable process as alive even when the direct link between the two processes is faulty.

## Outsourced Heartbeats

- Processes can outsource the task of sending heartbeats to their neighbors.
- This prevents the need for each process to be aware of all other processes in the network. They only need to be aware of a subset of connected peers.

## Phi-Accural Failure Detector

- Instead of using a binary scale (0/1), phi-accrual failure detector uses a continuous scale, capturing the probability of the monitored process’s crash.
- It collects arrival times of the most recent heartbeats from the peer processes.
- It uses the collected information to make a reliable judgment about node health.

φ = phi == how likely we can make correct decision about a process’s liveness == how likely we can make a mistake and receive a heartbeat that will contradict the calculated assumptions.

## Gossip and Failure Detection

- Each process maintains 3 things:
  - a list of neighbors
  - their heartbeat counters
  - timestamps specifying when the heartbeat counters were last incremented
- Each process periodically selects a random neighbor and sends it a heartbeat message.
- The receiving process increments the counter for the sender in his list of neighbors.

If any node did not update its heartbeat counter for a long time, it is considered dead.

## Reversing Failure Detection Problem Statement

Problem so far:
- Propagating information about failures is not always possible.
- Propagating it by notifying all processes is expensive / not efficient.

FUSE is one of the approaches to implement a failure notification service.
- It focuses on reliable and cheap failure propagation.
- Works even when the network is partitioned.

How it works?
- We arrange all active processes in groups.
- If one of the groups becomes unavailable, all participants detect the failure. In other words, if a single process failure is detected, it is converted and propagated as a group failure.

This approach allows detecting failures in the presence of any pattern of disconnects, network partitions, and node failures.

Within the group:
- Processes send ping messages to each other periodically.
- If a process does not receive a ping message from a peer for a long time, it stops responding to ping messages itself.

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.