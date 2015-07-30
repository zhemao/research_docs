# A Proposal for Supporting Memory-Mapped IO in RocketChip

In the current rocket core, memory-mapped IO is impossible since the core
assumes that all memory requests are cacheable. In order to remove this
limitation, I propose making the following modifications to RocketChip.

1. Add support for loads and stores that bypass the L1 data cache and L2 cache
   to read/write directly from the memory bus.
2. Replace RocketChip's top-level MemIO port with an AXI4 port.

## Configuration File Changes

I suggest adding two new configuration parameters to support memory-mapped IO.
The first is an integer "MMIOBase" which marks the boundary between regular
memory and IO memory. Physical addresses greater than MMIOBase will be treated
as IO memory and will not be cached.

The second is an AddressMap parameter, which includes all the information
about how the memory space should be partitioned for different peripherals.
This includes a unique identifier for the peripheral, the base address, and
a address space size. This information will be used to generate the AXI4 bus.

We have two options for handling the starting address. We could either require
the starting address to be explicitly written in the AddressMap parameter,
or we could simply take the sizes and automatically generate the starting
addresses. The latter would be less error prone, but we would need a way of
displaying the final address map. We might also allow both explictly set
starting addresses and auto-generated ones by having the second element of the
tuple be an option.

    case MMIOBase => 512 * 1024 * 1024 // DRAM is 512 MB
    case AddressMap => Seq(
        ("mem", Some(0x0), site(MMIOBase)), // 512 MB DRAM
        ("csr", Some(site(MMIOBase)), 4096), // 12-bit CSR space
        ("test", None, 16)) // test peripheral

There will probably need to be further additional options for the NASTI-related
modules in the junctions module. For instance, the width of the address field
should be different for different slave interfaces, so we will need to do
some sort of name-based secondary lookup similar to what we currently do for
cache parameters.

## Bypassing L1 Cache

Modifying the L1 cache to allow memory requests to bypass the cache will be
rather difficult. Currently, the L1 cache uses a non-blocking architecture
with several miss status holding registers (MSHRs). The MSHRs wait for a
request to come back from the next-level cache, store the result in the data
array, and then replays the memory request from the CPU.

                |-------------|
                | CPU Request |
                |-------------|
                       |
                       |
                 |-----------|
                 | Tag Match |
                 |-----------|
                Hit |    | Miss
                    |    |
                -----    --------
                |               |
                |        |------------|
                |        | L2 Request |
                |        |------------|
                |               |
                |     |-------------------|
                |     | Update Data Array |
                |     |-------------------|
                |               |
                |           -----
                |           |
            |-------------------|
            | Access Data Array |
            |-------------------|
                      |
             |-----------------|
             | Respond to User |
             |-----------------|

In this scheme, the response sent to the user will always come out of the
data array. However, if we are to support uncached loads and stores,
we will have to allow the data to come directly from the response from the
next-level cache.

                |-------------|
                | CPU Request |
                |-------------|
                       |
                       |
                 |-----------|
                 | Tag Match |
                 |-----------|
                Hit |    | Miss
                    |    |
                -----    --------------------
                |                           |
                |                    |------------|
                |                    | L2 Request |
                |                    |------------|
                |               Cached |       | Uncached
                |               --------       |
                |               |              |
                |     |-------------------|    |
                |     | Update Data Array |    |
                |     |-------------------|    |
                |               |              |
                |           -----              |
                |           |                  |
            |-------------------|              |
            | Access Data Array |              |
            |-------------------|              |
                      |                        |
                      -----            ---------
                          |            |
                       |-----------------|
                       | Respond to User |
                       |-----------------|

There are two ways we could go about changing the L1 cache architecture to
support this functionality. The first way would be to change the MSHR design to
allow it to fulfill both cached and uncached requests. The other way is to
leave the current MSHR design as is and design a special MSHR specifically
for handling uncached operations.

The benefit of having a seperate MSHR for uncached access is simplicity.
The control flow for an uncached access versus a cached access diverges at the
very beginning. Instead of picking apart the current MSHR controller, it would
be much less error-prone to build up a new controller from scratch.
Also, the resources needed for an uncached access are much smaller
than that for a cached access. An uncached access does not need to access the
data and metadata tables, consider cache coherence, or maintain a replay queue
for handling secondary misses.

The downside, of course is a loss in flexibility. The MSHR file will need to
include two different types of MSHRs, each of which can only handle a specific
type of access. We might run into a load-balancing issue in which the caching
MSHRs sit idle while the non-caching MSHRs are heavily loaded.

Regardless of which scheme, we use, there is one other modification we need
to make, which is to change the path through which responses are sent to the CPU.
Currently, the responses always come from the data array. However, to support
uncached accesses, we need a way for the responses to come directly from the
MSHRs. One way to do this is to have a response queue and have an arbiter
between the queue and the data array.

## AXI4

In place of the MemIO interface, the top-level module will export an AXI4
interface. Internally in RocketChip will be an AXI4 interconnect, which will
connect several AXI masters to AXI slaves. The AXI masters will include one
from each core, as well as an external interface to allow access from the host.
This external master interface will replace the current HTIF interface.

The slaves will be the main memory, a CSR memory map for each core, and
additional peripheral devices. Only the main memory will be exported directly
as AXI. The CSR memory maps will go to internal adapters. The peripheral
devices will probably map to some other interface (like SPI, I2C, Ethernet, etc.).
