# Stateful firewall



## State Table Organization

The connection table is implemented as a three-dimensional associative memory indexed as:

```text
Shard → Row → Associative Entry
```

Each connection entry stores:

* Source IP
* Destination IP
* Source Port
* Destination Port
* Protocol
* Connection State
* TCP Sequence Number
* TCP Acknowledgement Number
* Expiration Time

A separate `nextWriteIndex` memory maintains the insertion pointer for every row, implementing round-robin replacement across the associative entries.

---

## Hash Function Review

The `lightweight_hash()` function computes a 32-bit hash using the packet's source IP, destination IP, source port, destination port, and protocol number.

Before hashing, the function orders the IP addresses and ports so that packets belonging to the same TCP connection generate the same hash regardless of packet direction. This avoids maintaining separate hash entries for forward and reverse traffic.

The generated hash is divided into:

* Lower 16 bits → shard selection
* Upper 16 bits → row selection

The indices are reduced using the modulus operator to fit the configured number of shards and table depth.

---

## Connection Lookup

Connection lookup is encapsulated inside the `connection_match()` function.

A connection is considered valid only when:

* The stored state is not `INVALID`.
* The connection has not expired.
* Source and destination IP addresses match.
* Source and destination ports match.
* Protocol matches.



## Stateless Filter

The stateless filter is implemented as

```text
(hash % 100) >= 10
```

Only new TCP SYN packets pass through this filter before being inserted into the connection table. Packets failing this condition are rejected without creating a state entry.

---

## Packet Classification

The combinational preprocessing block derives packet categories directly from the TCP control flags.

The implementation identifies:

* Pure SYN
* Pure ACK
* Pure FIN
* SYN+ACK
* FIN+ACK
* Data Packet

These intermediate signals simplify the sequential processing logic by avoiding repeated flag comparisons.

---

## Sequential Packet Processing

The sequential block executes on every positive clock edge.

During reset, every state table entry is initialized to zero or `INVALID`, and all output signals and counters are cleared.

For every valid packet, the module:

* Clears previous status outputs.
* Computes lookup locations.
* Checks for illegal TCP flag combinations.
* Processes only TCP packets.
* Performs connection lookup.
* Updates connection state.
* Generates a single packet result.

---

## Connection State Handling

The implementation supports the following connection states:

* INVALID
* NEW
* ESTABLISHED
* FIN_WAIT
* CLOSE_WAIT
* TIME_WAIT
* RELATED

State transitions are driven by the received TCP packet type.

RST packets invalidate matching entries.

SYN+ACK packets transition a connection from `NEW` to `ESTABLISHED` after acknowledgement validation.

ACK and data packets refresh timeout information and update stored sequence and acknowledgement numbers.

FIN and FIN+ACK packets transition the connection through `FIN_WAIT`, `CLOSE_WAIT`, and `TIME_WAIT` depending on the current stored state.

---

## Sequence Number Validation

Before accepting packets belonging to an existing connection, the implementation validates TCP sequence numbers.

SYN+ACK packets require the acknowledgement number to equal the stored sequence number plus one.

ACK, data, FIN, and FIN+ACK packets verify sequence numbers against the stored value and update the stored sequence and acknowledgement numbers when newer values are received.

---

## New Connection Insertion

If no existing connection is found and the packet is a pure SYN, the module inserts a new entry after passing the stateless filter.

The insertion procedure:

* Selects the associative entry indicated by `nextWriteIndex`.
* Stores all connection parameters.
* Initializes the state to `NEW`.
* Stores sequence and acknowledgement numbers.
* Initializes the expiration timer.
* Advances the round-robin pointer.
* Increments the `newConns` counter when appropriate.

---

## Testbench Review

The testbench verifies the firewall using 46 TCP packets stored in an external memory file loaded through `$readmemh`.

For each packet, the testbench:

* Loads all packet fields.
* Drives the packet into the DUT.
* Prints packet information.
* Monitors the packet result.


The testbench also verifies that exactly one output (`packet_allowed`, `packet_dropped`, or `packet_rejected`) is asserted for every packet and reports any unknown or multiple-result conditions.

---

