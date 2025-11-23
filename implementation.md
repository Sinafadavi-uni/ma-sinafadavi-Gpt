\section{Architectural plan of datastore}

Below is a complete architecture for the datastore system. It is based only on three key strategies:

\begin{itemize}
    \item Storing multiple copies of data for safety,
    \item Using a decentralized method to find where data is stored,
    \item And repairing differences between replicas quickly and efficiently using Merkle trees.
\end{itemize}
This design ensures reliability, scalability, and low network usage, even when nodes fail or go offline.

The datastore system is a decentralized key–value store that replicates data across multiple nodes. It has no single point of failure because each piece of data is stored on more than one node. Keys are assigned to nodes using consistent hashing, so there is no need for a central coordinator. When replicas become inconsistent, the system uses Merkle trees to find and fix differences efficiently.

\subsection{Core strategies integrated}

\begin{itemize}
    \item \textbf{Multiple redundant copies (no single point of failure)}
\end{itemize}

Each key–value pair is stored on more than one node. The system uses a replication factor of at least two (\(r \geq 2\)), ensuring that multiple copies are always maintained.



When a write happens, all replicas are updated at the same time. The write only succeeds if all replicas confirm it.
This means if one node fails or goes down, the data is still available on the other nodes. There is no single point of failure.

\begin{itemize}
    \item \textbf{Decentralized data mapping (no central coordinator)}
\end{itemize}

The nodes are organized in a ring using consistent hashing.
Any node or client can figure out which nodes are responsible for a given key: it just hashes the key and looks at the ring to find the correct place by moving clockwise.
Because of this, there’s no need for a central directory or metadata server. Nodes can join or leave the system without requiring changes across the whole cluster.

\begin{itemize}
    \item \textbf{Efficient incremental repair using Merkle trees}
\end{itemize}

Each node builds a Merkle tree that represents its stored data. This tree acts like a summary.
Replicas regularly exchange their root hashes. If the roots are different, they go deeper into the tree to find which parts don’t match. Then, only the keys and values from those mismatched sections are transferred.
This process, called anti-entropy, keeps all replicas in sync with very little data sent over the network. It makes repairs fast and efficient.



\begingroup
  \setlength{\parskip}{1em}   % or some length you like
  \setlength{\parindent}{0pt} % optional: remove indentation in this block








\section{Design overview}

\subsection{Decentralized data mapping }
Lays the routing substrate every other replication feature depends on; without it, we don’t know where replicas belong or how to reach them.

\begin{itemize}
    \item \textbf{Core responsibility}
\end{itemize}
The system’s main job is to decide where data should be stored which node gets which key without needing a central coordinator. It does this in a predictable and reliable way using consistent hashing.


\begin{itemize}
    \item \textbf{Key modules}
\end{itemize}

\textbf{membership\_manager:} This part keeps track of which nodes are currently active, whether they’re healthy, and how virtual nodes (tokens) are assigned across the ring.

\textbf{hash\_ring:} This is the implementation of consistent hashing. When given a key, it produces an ordered list of nodes that should store it.

\textbf{routing\_client:} A small library used by brokers, executors, and datastores. It lets any node find out which replicas are responsible for a given key by using the current hash ring.

\begin{itemize}
    \item \textbf{Data flow}
\end{itemize}

When a node joins or leaves the network, it sends a message about this change. The membership manager picks up this event and updates the hash ring accordingly.
Other nodes eventually receive this updated ring state. Then, whenever a client needs to read or write a key, it uses its local copy of the ring to compute the correct set of target nodes all locally, with no need to ask a central server.
This means reads and writes go directly to the right place, fast and without a central directory.


\begin{itemize}
    \item \textbf{Contracts}
\end{itemize}

- The function f(k) takes a key k and returns an ordered list of N nodes that are responsible for storing it.

- Hash ring snapshots include version information so different nodes can detect outdated views and gradually converge to the same state over time.


\begin{itemize}
    \item \textbf{Coding steps}
\end{itemize}


First, we define the data structures for the hash ring. This includes node tokens (positions on the ring), support for virtual nodes to balance load better, and version numbers for the ring itself so changes can be tracked. We also decide how this data is saved and shared between nodes using a simple format like JSON or MessagePack.

Next, we implement a gossip-based membership system where nodes send heartbeat messages to share their status. The membership manager collects this information and uses it to build a list of live nodes. Based on that list, it assigns tokens in a consistent and predictable way, so every node calculates the same assignments given the same set of active nodes.

We then write a pure function one that always gives the same output for the same input that takes a key and returns the list of nodes responsible for storing it. This function is used by all components. We also add helper methods to find the next nodes in the ring, which helps during replication when writing data to multiple replicas.

The routing client is updated to use this local ring logic. Brokers, executors, and datastores now compute where data should go on their own, without asking a central server. This replaces any old lookup mechanism that depended on a coordinator.


\textbf{To help monitor the system, we add observability features:}

- A ring checksum to quickly check if nodes have similar views

- Metrics that show how often keys are remapped when nodes join or leave

- Warnings if different nodes see conflicting ring versions

\subsection{Multiple redundant copies }
Builds on the mapping layer to guarantee availability by placing and tracking replicas across the ring.

\begin{itemize}
    \item \textbf{Core responsibility}
\end{itemize}
The system makes sure that every piece of data (each key) is stored on at least N different nodes, so it won’t be lost if one node fails. But it also respects network limits especially important in low-bandwidth environments by not copying too much or too often.

\begin{itemize}
    \item \textbf{Key modules}
\end{itemize}

\textbf{replication\_controller:} This runs inside brokers or datastores and manages how data is copied. It decides which nodes should store a given key, sends out the write requests, and keeps track of which replicas have responded.

\textbf{replica\_metadata\_store:} A small database (possibly kept at the broker) that stores information about each key’s replicas  like their current state, version numbers, and how many responses are needed for success (quorum).

\textbf{consistency\_policy:} A flexible layer that defines how strong consistency should be. For example, it sets values like W (how many replicas must confirm a write) and R (how many to read from), and supports features like 
 to handle temporarily offline nodes.

\begin{itemize}
    \item \textbf{Data flow}
\end{itemize}
When a client sends a write request, it goes to a coordinator node (like a broker or a designated datastore). The replication controller looks up the key in the hash ring and gets a list of N preferred nodes (the preference list). It sends the write to all of them, but only waits for W acknowledgments before saying the write succeeded.
Once confirmed, it updates the replica metadata. If some replicas were unreachable, their pending writes are added to a hinted handoff queue and sent later when they come back.

\begin{itemize}
    \item \textbf{Contracts}
\end{itemize}

A write is only considered durable once it receives the required number of successful replies (based on W).
Each replica has a state such as fresh, syncing, stale, or tombstoned that helps the system know what needs repair later.
For executor results, the “first valid result wins” rule still applies. But now, the system uses replica metadata to check if a result comes from an up-to-date copy before accepting it.

\begin{itemize}
    \item \textbf{Coding steps}
\end{itemize}


We start by defining the replication settings, like the default number of copies (N), how many confirmations are needed for a write to succeed (W), and how many replicas to read from (R). These values can be changed later, so we make them part of a config service that’s easy to update.

Next, we extend the datastore interface with a put function that takes not just a key and value, but also a version and an origin ID. This helps track updates using version vectors, so we can handle concurrent changes. The operation is designed to be idempotent meaning doing it more than once doesn’t cause problems which is important when retries happen.


In brokers and datastores, we build the write pipeline:

First, we use the hash ring to find the list of N target nodes (the preference list).


Then we send the write to all of them at the same time.
Wait only for W successful replies before marking the write as complete.
If some nodes don’t respond, their pending writes go into a hinted handoff queue for later delivery.
We also add a background process a reconciler  that checks this queue regularly. When a previously offline node comes back online, the system tries to deliver the missed writes.

For reading, we update the read path so clients don’t just pick any replica. Instead, they check the replica metadata and choose the one with the most up-to-date version. This sets the stage for read repair fixing inconsistencies when they’re found.

Finally, we write tests to make sure everything works under real-world conditions.

\textbf{These include:}

- Simulating node failures

- Checking that quorum rules are respected

- Testing if hinted handoff correctly replays missed writes

- We also run chaos-style scenarios based on the design doc like killing a broker mid-write or creating network partitions  to see how well the system recovers

\subsection{Efficient incremental repair with Merkle trees }

\begin{itemize}
    \item \textbf{Core responsibility}
\end{itemize}


Once data is replicated across nodes, the system must make sure all copies stay in sync over time. The goal is to detect when replicas have become different (due to failures or network issues) and fix them but without sending large amounts of data. Only the missing or outdated parts should be transferred.

\begin{itemize}
    \item \textbf{Key modules}
\end{itemize}

\textbf{merkle\_tree\_builder:} This runs on each node and builds a Merkle tree for its stored data. The tree covers key ranges that match the consistent hash ring’s partitions, so every node structures their trees the same way.

\textbf{anti\_entropy\_scheduler:} This part decides when and which two replicas should compare their data. It looks at how long it’s been since they last synced and picks pairs that are likely out of date.

\textbf{repair\_executor:} When differences are found, this component fetches only the keys or chunks that don’t match and applies the necessary updates to bring the replica up to date.


\begin{itemize}
    \item \textbf{Data flow}
\end{itemize}

The repair process starts when the scheduler picks two replicas to check. They exchange the root hash of their Merkle trees. If the hashes don’t match, they go down into the tree step by step to find the exact subtree where the difference lies. Then, they request the specific list of keys that differ. Only those keys are sent from the newer copy to the older one. Once the update is applied, the replica’s state is updated to “healthy,” showing it’s back in sync.

\begin{itemize}
    \item \textbf{Contracts}
\end{itemize}

Merkle trees are built over fixed data ranges that match the hash ring’s partitioning, so all nodes agree on what each tree covers.

The same hash function and tree structure (like branching factor) are used everywhere, so trees can be compared correctly between different nodes.

After a repair session, the replica metadata is updated. This helps future reads know which copies are fresh and reliable.

\begin{itemize}
    \item \textbf{Coding steps}
\end{itemize}
We design the Merkle tree system as a reusable module with three main functions:

\textbf{generate:} builds a tree from a snapshot of a data segment

\textbf{serialize:} turns the tree into a format that can be sent over the network

\textbf{diff:} compares two trees to find differences
The trees are built on stable chunks of data that match how the hash ring divides up keys, so every node processes the same ranges.

Tree generation runs in the background on each datastore, so it doesn’t slow down regular reads and writes. We add throttling to limit how much bandwidth it uses important in low-resource environments.

Next, we implement the anti-entropy protocol:

Two replicas start a repair session and agree on which key range to check. They exchange the root hash of their Merkle trees. If they don’t match, they go deeper comparing child nodes step by step until they find the exact key-level differences. This process handles empty or missing ranges gracefully, avoiding errors when one side has no data.

When mismatches are found, the repair executor kicks in. It only requests the keys that are missing or outdated. Before applying updates, it checks version numbers and uses rules like last-write-wins or vector clock comparison to resolve conflicts safely.

After repair, the system updates the replica’s metadata:

- It marks the replica as up-to-date

- It clears any pending hinted handoff entries for that node, since the data is now current. 

This helps keep the system clean and avoids unnecessary retries.

Finally, we write simulation and integration tests that create realistic problems like dropped updates or stale replicas and then check that:

- The repair process fixes them correctly

- Only small amounts of data are transferred (not full copies)

- Normal operations like reading and writing continue without being blocked during repair



\subsection{Summary}


We build the system step by step. First, we set up decentralized routing so any node can find where data should go no central server needed. Then, we add quorum-based replication, which means each piece of data is stored on multiple nodes, and we only need a certain number of them to respond for reads and writes to succeed. This gives us durability without requiring every node to be online. Finally, we add Merkle tree-based repair (anti-entropy) to automatically fix differences between replicas, but only sync what’s missing saving bandwidth.

This step-by-step approach turns the architecture into working code in a clear way. Each part creates reusable components and shared metadata that the next part can use. Because everything is built and tested in stages, the risk stays low, and we can check each piece works before moving on.

The result is a system that stays reliable even when nodes fail, doesn’t waste network resources, and has no single point of failure  perfect for edge environments where connections are weak or unstable.

