# Chapter 14: Consensus

Consensus == Multiple processes reaching agreement on a value.

Correct process == A process that hasn't crashed and is executing its algorithm.

3 properties of consensus algorithms:
1. Agreement: All correct processes decide on the same value.
2. Validity: The value decided upon is proposed by some process.
3. Termination: All correct processes eventually decide on a value.

## Broadcast

Broadcast == A process sends a message to all other processes.

- Reliable broadcast == All correct processes receive the message.
- Naive version of reliable broadcast == All processes send the message to all other processes.
- Downside of naive version:
  - Network is flooded with messages, since this requires N^2 messages, where N is the number of processes.
  - Order of messages is not guaranteed.

## Atomic Broadcast

Atomic broadcast guarantees 2 properties:
- Atomicity: Either all correct processes deliver the message, or none do.
- Order: If a correct process delivers message m1 before message m2, then all correct processes deliver m1 before m2.

## Virtual Synchrony

Atomic broadcast delivers messages to a static group of processes. Virtual synchrony extends this to dynamic groups.

Virtual synchrony:
- Ordered Message Delivery: Delivers totally ordered messages consistently to all group members.
- Group Organization: Processes organized into groups with shared views, ensuring consistent order.
- Dynamic Group Changes: Group views change with join, leave, or failure announcements, affecting message association.
- Atomic Delivery Guarantees: Ensures atomic message delivery between view changes, serving as barriers. Limited adoption in end-user systems.

## Zookeeper Atomic Broadcast (ZAB)

ZAB is a popular implementation of atomic broadcast. It is used in Zookeeper, a hierarchical distributed key-value store.

Processes in ZAB can take one of the following roles:
1. Leader: Drives the process, establishes event order, and manages algorithm execution in ZAB.
2. Follower: Listens to the leader, follows broadcasted messages.

After leader is established, protocol executes in 3 phases:
1. Discovery: Leader learns the latest epoch from followers, proposes a new epoch, and obtains follower responses regarding the latest transaction in the previous epoch.
2. Synchronization: Leader proposes itself as a leader, collects acknowledgments from followers, and ensures synchronization, delivering committed proposals from previous leaders before any new epoch proposals.
3. Broadcast: Leader initiates active messaging, establishes message order, and broadcasts client messages by proposing, waiting for follower acknowledgments, and committing.

Advantages of ZAB:
- Efficiency: Broadcast processes requires only 2 rounds of messages.
- Fault Tolerance: Leader failure is handled by electing a new leader.

## Paxos

Paxos == Most widely known consensus algorithm.

Participants in Paxos can take the following roles:
1. Proposer: Proposes a value to acceptors and collects votes.
2. Acceptor: Votes for a value proposed by a proposer.
3. Learner: Takes the role of replicas, and stores the outcomes of accepted proposals.

## Paxos Algorithm

Paxos algorithm executes in 2 phases:
1. Propose/Voting Phase: Proposer sends a proposal to acceptors, who vote for the proposal. If a proposal receives a majority of votes, it is accepted.
2. Replication Phase: Proposer distributes the accepted value to acceptors.

Acceptor has to follow these invariants when responding to a proposal with Prepare(n) message, where n is the proposal number:
1. If acceptor has not responded to any Prepare(m) message (where m > n), it promises not to respond to any proposal with a number less than n.
2. If acceptor has already accepted any other proposal having proposal number m earlier, it notifies the proposer that it has already accepted a proposal with a proposal number m.
3. If acceptor has already responded to Prepare(m) message (where m > n), it notifies the proposer about the existence of a higher proposal number.
4. Acceptor can respond to more than one Prepare(x), as long as the later proposals have a higher proposal number.

✨ **Side learning:** Very good explanation of Paxos - https://www.youtube.com/watch?v=SRsK-ZXTeZ0

## Quorums in Paxos

Quorum == The minimum number of votes required for the operation to be performed.

Generally, quorum == majority of participants.

Once sufficient number of participants (quorum) have accepted the proposal, it's guaranteed that the value will be accepted, since any two majorities will have at least one participant in common.

To guarantee "liveness", the protocol requires 2f+1 participants, where f is the number of failures that can be tolerated.

## Failure Scenarios

**Case 1: Proposer fails during 2nd phase, before broadcasting value to all acceptors**

A new proposer would pick up the value from quorum and broadcast it to all acceptors.

- If the value received from quorum is the same as proposed by original proposer, the new proposer will continue with the broadcast using the same value.
- If the value received from quorum is different from the one proposed by original proposer (and the new proposer doesn't know this), the new proposer will commit the value received from quorum, which is not the value proposed by original proposer.

**Case 2: When 2 or more proposers start competing**

- To avoid multiple proposers failing to collect a majority, a random backoff time can be used for each proposer to wait.

TLDR: Paxos can tolerate acceptor failures, but only if there are still enough acceptors alive to form a majority.

## Multi-Paxos

- Problem with Paxos: It requires 2 phases for every proposal, which can be slow.
- Solution: Multi-Paxos, where a single proposer can propose multiple values in a single round.
- To avoid repeating the "propose" phase, multi-paxos has a leader (effectively a distinguished proposer) that can propose values without going through the "prepare" phase.

## Fast Paxos

- Fast Paxos is an optimization of the original Paxos algorithm designed to reduce the number of message rounds required for reaching consensus.
- This is done using "fast rounds", when acceptors can proceed with messages from proposers other than the established leader.

✨ **Side learning:** [Good short blog on Fast Paxos](https://medium.com/@hrjeet0987/fast-paxos-1a0923a97d95)

## Egalitarian Paxos

Problems with distinguished proposer as a leader:
- Makes system prone to leader failures.
- Dispropotionate load on leader, impairing system performance.

Solution:
- Instead of using leader and proposal numbers to order proposals, we use a leader that is responsible for the commit of the specific command, but not for the ordering of the commands.
- The event order is established by resolving dependencies between submitted messages.

"Dependencies" == All commands that interfere with a current proposal.

## Flexible Paxos

- Instead of requiring quorum for both phases, Flexible Paxos relaxes the quorum requirements.
- It only requires quorum for the 1st phase (voting) to intersect with a quorum for the 2nd phase (replication).
- Flexible Paxos allows trading availability for latency.
  - We reduce the number of nodes required for the 2nd phase, which reduces the latency.
  - But we have to collect more votes in the 1st phase, requiring more participants to be available in the 1st phase.

For example:
1. Let's say we have 5 acceptors.
2. Let's say we require collecting votes from 4 acceptors to accept a proposal.
3. Then we only need to wait for responses from 2 nodes during the replication phase, since it's guaranteed that atleast one of the nodes that voted for the proposal will be present in the 2 nodes that we are waiting for.

## Generalized Solution to Consensus

I feel this section is a bit too theoretical, and is rushed in the book. So instead of trying to create notes, I'd recommend checking out the following blog: https://blog.acolyer.org/2019/03/08/a-generalised-solution-to-distributed-consensus/

## Raft

Raft == A consensus algorithm designed to be easy to understand.

Participants in Raft can take one of the following roles:
1. Candidate: Proposes itself as a leader.
2. Leader: Temporary leader, that handles client requests and replication.
   - Leader is elected for a period called "term".
   - "Term" is identified by a monotonic increasing number.
3. Follower: Responds to requests from leaders and candidates. Persists log entries.

**How is global partial order maintained in Raft?**
- Time is divided into "terms" / "epochs".
- Each command is uniquely identified by 2 things:
   1. the term number
   2. an index (message number) within the term.

Main components of Raft algorithm:



1. Leader Election:
   1. Candidate P1 initiates by sending RequestVote with term, last known term, and last log entry observed.
   2. Majority votes lead to successful election, ensuring each process can vote for at most one candidate.
2. Periodic Heartbeats:
   1. Leader sends periodic heartbeats to followers for liveness.
   2. If a follower doesn't receive heartbeats within an election timeout, it assumes leader failure and triggers a new election.
3. Log Replication/Broadcast:
   1. Leader appends new log entries using AppendEntries messages.
   2. Message includes leader’s term, index, term of the preceding log entry, and one or more entries for replication.

## Leader Role in Raft

- Election Criteria:
  - Leader elected from nodes with all committed entries.
  - Follower denies the vote if its log is more up-to-date.
- Leadership:
  - Leader replicates client requests, appends entries to log.
- Follower Response:
  - Followers append entries, lets leader know that it was persisted.
  - As soon as enough replicas send their acknowledgments, leader commits the entry.
- Log Flow:
  - Unidirectional flow from leader to follower, because only the most up-to-date candidate can be elected as leader.

## Failure Scenarios

Scenarios:
- Split Vote: Multiple followers decide to become candidates, and no candidate gets majority votes.
  - Randomized timers are used to reduce probability of split vote.
- Followers may be down or slow to respond, impacting message delivery.
  - Leader retries message delivery and optimizes performance by sending multiple messages in parallel.
- Leader may fail, and a new leader may be elected.
  - Newly elected leader restores cluster state to the last known up-to-date log entry.

## Byzantine Consensus

All algorithms discussed so far assume that participants are honest. Byzantine consensus algorithms are designed to tolerate malicious participants.

Byzantine algorithms require extensive N^2 messages for cross-validation.
- This is because nodes cannot rely on the messages from other nodes, and have to validate the received message by comparing it with the message received by majority of the nodes.

## PBFT Algorithm

Practical Byzantine Fault Tolerance (PBFT): It is a Byzantine consensus algorithm that tolerates f failures in a network of 3f+1 nodes.

**Cluster Configurations with "Views"**
- Views distinguish between configurations.
- Each view has a "primary" and remaining nodes are "backups".
- Primary executes client operations, broadcasting requests to backups.
- Client waits for 2f + 1 matching responses from backups for any operation to succeed.

**Protocol Phases**
- Pre-prepare: Primary node broadcasts view ID, unique identifier, payload, and payload digest.
- Prepare: Backups broadcast Prepare messages with view ID, message ID, and payload digest, validating against pre-prepare.
- Commit: Backups broadcast Commit messages, waiting for 2f + 1 matching Commit messages for execution.

**Recovery and Checkpointing**

- Replicas store messages until executed by at least f + 1 nodes.
- Nodes compute a digest for state verification during recovery.
- Primary makes stable checkpoints after N requests, waits for 2f + 1 replica responses.

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.
