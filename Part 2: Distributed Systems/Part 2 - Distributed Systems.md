# Part 2: Distributed Systems

Vertical scaling == increasing the resources of a specific node.

Horizontal scaling == adding more nodes to the system.

Vertical scaling is limited by the hardware of the machine, so we need to scale horizontally.

## Basic Definitions

- Participants in a distributed system == nodes, replicas, processes.
- Participants communicate with each other via messages using communication links between them.
- Processes access time using a clock (logical or physical).

Purposes of distributed algorithms:
- Coordination: Supervise the actions of several workers.
- Cooperation: Multiple participants have to rely on one another to complete a task.
- Dissemination: Distribute information to all interested participants quickly and reliably.
- Consensus: Achieving agreement among a number of processes.
