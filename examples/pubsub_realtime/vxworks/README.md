# Open62541 pubsub VxWorks TSN publisher

This TSN publisher is a complement to [pubsub_interrupt_publish.c](../pubsub_interrupt_publish.c)(For more information, please refer to its [README](../README.md)). It is a VxWorks specific implementation with VxWorks-native TSN features like TSN Clock, TSN Stream, etc, which can be applied to hard realtime scenarios.

In this publisher, TSN cycle is triggered by TSN Clock -- A high-precision hardware interrupt generated by a 1588 device. The ISR will wake up a task to publish OPC-UA packets, one packet per cycle.

Considering in a typical VxWorks user scenario -- an embedded device with limited compute resources and probably without local storage, the performance will not be measured directly by this publisher. Instead, the publisher will collect all the necessary data, pack the data into an OPC-UA packet and publish it.

The measurement data includes:

1. The trigger time(1588 time) of each cycle.
2. Each wake-up time(1588 time) of the publisher task.
3. Each end time(1588 time) of the publisher task.

## User interfaces

This publisher provides 2 APIs to users:

1. `open62541PubTSNStart`

   ```C
   /**
    * Create a publisher with/without TSN.
    * eName: Ethernet interface name like "gei", "gem", etc
    * eNameSize: The length of eName including "\0"
    * unit: Unit NO of the ethernet interface
    * stkIdx: Network stack index
    * withTsn: true: Enable TSN; false: Disable TSN
    *
    * @return OK on success, otherwise ERROR
    */
    STATUS open62541PubTSNStart(char *eName, size_t eNameSize, int unit, uint32_t stkIdx, bool withTsn);
   ```

   In order to get the best performance, this API supports to associate a dedicated CPU core with TSN related interrupt and tasks. The stkIdx argument is used for this purpose. Its value is the CPU core number: 0 means Core 0, 1 means Core 1, etc.

2. `open62541PubTSNStop`. This API is used to stop the publisher.

   ```C
   void open62541PubTSNStop();
   ```

## Building the VxWorks TSN Publisher

This publisher should be built under VxWorks' building environment. For specific building instructions, please refer to VxWorks' user manual.

## Running the VxWorks TSN Publisher

1. Before starting the publisher, you should make sure that PTP is well synchronized. For the usage of PTP, please refer to VxWorks' user manual.
2. Do TSN configuration by tsnConfig as below:

   ```sh
   -> tsnConfig("gei", 4, 1, 0, "/romfs/62541.json")
   ```

   `62541.json` is a TSN configuration file. Below is one example configuration using 250 usecs cycle time. Adjust cycle_time and tx_time according to your system performance.

   ```json
   {
    "schedule": {
        "cycle_time": 250000,
        "start_sec": 0,
        "start_nsec": 0
    },
    "stream_objects": [
        {
            "stream": {
                "name": "flow1",
                "dst_mac": "01:00:5E:00:00:01",
                "vid": 3000,
                "pcp": 7,
                "tclass": 7,
                "tx_time": {
                    "offset": 200000
                }
            }
        }
    ]
    }
   ```

3. Then run `open62541PubTSNStart`. Below is an example.

   ```sh
   -> open62541PubTSNStart("gei", 4, 1, 3)
   value = 0 = 0x0
   -> [1970-01-01 00:47:33.100 (UTC+0000)] warn/server     Username/Password configured, but no encrypting SecurityPolicy. This can leak credentials on the ne.
   [1970-01-01 00:47:33.100 (UTC+0000)] warn/userland      AcceptAll Certificate Verification. Any remote certificate will be accepted.
   [1970-01-01 00:47:33.100 (UTC+0000)] info/userland      PubSub channel requested
   [1970-01-01 00:47:33.100 (UTC+0000)] info/server        Open PubSub ethernet connection.
   [1970-01-01 00:47:33.100 (UTC+0000)] info/userland      Adding a publisher with a cycle time of 0.250000 milliseconds
   [1970-01-01 00:47:33.200 (UTC+0000)] info/network       TCP network layer listening on opc.tcp://vxWorks:4840/
   ```

4. Run `open62541PubTSNStop` to stop the publisher.

   ```sh
   -> open62541PubTSNStop
   [1970-01-01 00:48:16.500 (UTC+0000)] info/server        Stop TSN publisher
   [1970-01-01 00:48:16.516 (UTC+0000)] info/network       Shutting down the TCP network layer
   [1970-01-01 00:48:16.516 (UTC+0000)] info/server        PubSub cleanup was called.
   value = 0 = 0x0
   ```
