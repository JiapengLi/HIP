- Start Date: 2020-02-20
- HIP PR: <!-- leave this empty -->
- Tracking Issue: <!-- leave this empty -->

# Summary
[summary]: #summary

This HIP proposes a routing scheme for LoRaWAN packets post-join, via DevAddr. A
block of DevAddr addresses will be obtainable for an OUI via a blockchain
transaction. This range of addresses will be assigned to devices joined to this
OUI and hotspots will be able to do fast and simple 'range routing' to route
packets to their destination OUI. These address ranges should be transferrable
and splittable (down to some minimum size) to allow for address reconfiguration
in the future. A related HIP describes the routing of LoRaWAN join packets.

# Motivation
[motivation]: #motivation

The Helium network is a multi-tenant network, we envision multiple
organizations controlling their own OUIs and devices without any central
authority. Thus we need an unambiguous routing mechanism so that hotspots know
where to send device packets. This routing scheme should not require frequent
updates or extensive coordination.

The current strategy is to alias devices by assigning them the OUI as their
DevAddr and brute forcing the Message Integrity Code of the packet against all
known active session keys. While this is relatively fast (about 30 microseconds
on an Intel i7-7700HQ), an OUI with a very large number of active devices will
have to do this, as a linear search, a lot. We consider this will be a
scalability problem in the future.

# Stakeholders
[stakeholders]: #stakeholders

Anyone routing device traffic on the Helium network.

# Detailed Explanation
[detailed-explanation]: #detailed-explanation

- Introduce and explain new concepts.

- It should be reasonably clear how the proposal would be implemented.

- Provide representative examples that show how this proposal would be commonly
  used.

- Corner cases should be dissected by example.

We'd introduce new transactions to obtain/transfer/split 'address blocks',
similar to IP address space. These blocks should be available in some fixed
sizes that are neatly divisible into smaller sizes (eg powers of 2, as in IP
address blocks). The size of the allocation would affect the price required to
obtain them. Allocations would consist of {Start and Size} (eg {0, 512} or
{2048, 4096} etc). The blocks would be allocated contiguously, starting from 0,
serialized into a total ordering by the blockchain transaction order.

A new class of ledger entries would be added, in a new column family. These
entries would have the big-endian starting address as the key:
`<<Start:32/integer-unsigned-big>>` and the corresponding value would be the
owning OUI: `<<OUI:32/integer-unsigned-big>>`. Big endian is used because we
can take advantage of rocksdb's lexiographic sorting properties more effectively.

When a packet needs to be routed, the Hotspot simply needs to seek to the
DevAddr as big endian in the rocksdb column family. The iterator can then be
used to find the previous key. That key will be the start of the address
allocation, and the value will be the OUI to route to. This should be extremely
fast and efficent, and can be augmented with a cache if needed.

A range routing prototype has been added to the bloom_router repo and some
benchmarks have been done:

Intel i7-7700HQ
```
running with 1000 OUIs and 8000000 Devices
Attempting 1000 random lookups
Average memory 0.25Mb, max 0.29Mb, min -0.04Mb
Approximate database size 0.02Mb
Average lookup 0.004s, max 0.1s, min 0.00001s
Lookup misses 0
```

Raspberry Pi 4b
```
running with 1000 OUIs and 8000000 Devices
Attempting 1000 random lookups
Average memory 0.17Mb, max 0.19Mb, min 0.00Mb
Approximate database size 0Mb
Average lookup 0.04s, max 0.5s, min 0.00004s
Lookup misses 0
```

Raspberry Pi 3b+
```
running with 1000 OUIs and 8000000 Devices
Attempting 1000 random lookups
Average memory 0.08Mb, max 0.10Mb, min -0.06Mb
Approximate database size 0Mb
Average lookup 0.03s, max 0.4s, min 0.0001s
Lookup misses 0
```

The good news is that, on average, this is orders of magnitude faster, on
average, than the XOR routing scheme for joins, but it is still not free.

# Drawbacks
[drawbacks]: #drawbacks



# Rationale and Alternatives
[alternatives]: #rationale-and-alternatives

This is your chance to discuss your proposal in the context of the whole design
space. This is probably the most important section!

- Why is this design the best in the space of possible designs?

- What other designs have been considered and what is the rationale for not
  choosing them?

- What is the impact of not doing this?

# Unresolved Questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the HIP process
  before this gets merged?

- What parts of the design do you expect to resolve through the implementation
  of this feature?

- What related issues do you consider out of scope for this HIP that could be
  addressed in the future independently of the solution that comes out of this
  HIP?

# Deployment Impact
[deployment-impact]: #deployment-impact

Describe how this design will be deployed and any potential imapact it may have on
current users of this project.

- How will current users be impacted?

- How will existing documentation/knowlegebase need to be supported?

- Is this backwards compatible?

        - If not, what is the procedure to migrate?

# Success Metrics
[success-metrics]: #success-metrics

What metrics can be used to measure the success of this design?

- What should we measure to prove a performance increase?

- What should we measure to prove an improvement in stability?

- What should we measure to prove a reduction in complexity?

- What should we measure to prove an acceptance of this by it's users?