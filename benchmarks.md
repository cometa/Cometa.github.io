---
---
# Cometa 
###Performance Benchmarks

#### Author 
Marco Graziano 
Visible Energy, Inc.

####Date: 
August, 2015

#### Abstract
This document describes the methodology and the results of performance and scalability benchmarks of the Cometa server version 2.1.

----------


Table of Contents
------------------

[TOC]

----------

Introduction
--------
Cometa is an edge server mediating communication between mobile and Web applications and remote devices. Typical communication needs of a IoT device include responding to a message request from an application and sending event data to a server. Both paradigms are covered in the Cometa API.

Device Interaction Protocol
-----------------

#### Basic Concepts
At startup devices connect to Cometa using the `attach` HTTP POST method to establish a permanent connection to the Cometa server. Once a device is connected, Cometa provides the underlying transport for both message exchange that follows a typical  RPC (Remote Procedure Call) pattern, and to establish a data pipe used by the devices to send asynchronous events data upstream.

For interaction with devices Cometa follows the HTTP chunked transfer encoding format of the [HTTP 1.1 specifications](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html) in both directions.

#### Messages to Devices
Applications use the `send` HTTP POST method to invoke a RPC to a remote device, with the message body containing the request and the parameters. Once Cometa receives an RPC message for the device from the application, it relays the request to the device and expects a synchronous reply to the message. The reply received from the device is then relayed back to the requesting application as response to the RPC invocation.

A more performing method to send RPC invocation messages to devices is for an application to open the Cometa WebSocket for the device and just write and read to and from the WebSocket. Using a WebSocket avoids the overhead of establishing an HTTP connection for every RPC invocation.

#### Wire Protocol
A device attached to Cometa has a bi-directional transport for RPC and an upstream data pipe for events, but Cometa does not enforce any wire protocol, that is left to the application. For the benchmarks in this report no wire protocol has been adopted, with simulated devices responding to any request with a message of certain length.

Scalability of the Cometa Server
-----------------

Since the Cometa server has to maintain all the connections with the remote devices available all the time, it cannot be scaled horizontally with the use of a load balancer of reverse proxy which by definition do not establish long-lived connections. The implementation of the Cometa edge server must therefore allow for the largest possible number of simultaneous connections and still maintain a response time that does not degrade with a growing number of connected devices.

#### The 1024k Problem
The requirement to scale the edge server vertically poses several implementation challenges, which have been described in the literature as the [C10K problem](http://www.kegel.com/c10k.html) . The problem's name is a numeronym for "ten thousand clients". in our case we decided to tackle the C1024K “one million clients” problem instead, that is more in line with the requirements of a 21st century network infrastructure.

#### Multi-threading
The common approach in HTTP servers is to spawn a new thread for each client request, like in the Apache Web Server , the most popular web server on the Internet since 1996. In Cometa, since a connection must be maintained “forever”, this approach would quickly lead to a server resource overload because a significant amount of memory is allocated for each thread, and to CPU overload due to the thread context switching when a high number of request must be satisfied.

Cometa multi-threading takes advantage of multiple CPUs in modern servers and individual threads are each capable of managing a high number of concurrent device connections and requests. The number of threads, all capable of similar operations, rather than a function of connections, is more of a function of the number of CPUs of the server, and it is a configuration parameter. 

#### Asynchronous I/O
To allow for parallel processing of  simultaneous network requests in each thread, Cometa adopts asynchronous I/O (non-blocking) network communication. The ability for an application in the UNIX OS to use asynchronous communication relies on the paradigm that every resource in UNIX is a “file” and referred to with a “file descriptor”. This is also the case for Sockets used for TCP/IP style of communication needed for an HTTP connection that uses TCP.

The challenge when dealing with a large number of file descriptors and asynchronous I/O, is the strategy used in the UNIX kernel for checking individual file descriptor for data being available for reading, and to resume processing of that communication. It can be easily seen that if the kernel uses a simple scan or similar linear search strategy, it would operate in O(N) response time, where N is the number of open file descriptors (communication channels). Traditionally, UNIX kernels were only using this strategy in the `select` or `poll` system calls, while in modern UNIX systems other alternatives that operate in O(1) response time, are available. In Linux kernel 2.5.44 and subsequent releases, the `epoll`  kernel API for scalable I/O event notifications has been introduced, to achieve O(1) response time for more demanding applications, where the number of watched file descriptors is very large. 

The Cometa server is implemented in C language, and it makes use of the `epoll` kernel API for handling event-based, asynchronous I/O in exchanging data with the remote devices. With respect to memory requirements, it must be noted that in the device-server communication model used, no more than one UNIX Socket (communication channels) are used for each device at the same time. 

#### Scale Beyond C1024K

The results reported show the Cometa server is capable of supporting more than one million devices on a single 32 GB RAM, multi-core server and still provide millisecond latency in presence of high throughput rates. While servers with more than 32 GB RAM are widely available, once the capacity of one server has been reached it is still possible to deploy multiple edge servers. 

Operations are guaranteed with multiple edge servers provided that each has a different FQDN  and that clients are configured to connect to different Cometa server, in order not to exceed the capacity of a single server. This can become tricky when devices break down or are decommissioned by the owner.  Since configurable parameters can be remotely modified on devices in the field, the distribution of devices across many edge servers can be dynamically adapted as old devices are decommissioned. Such scenario must be contemplated in the device firmware at development time.


Benchmarking Kit
-----------------

#### Cometa Client Benchmark
The Cometa Client Benchmark `cb-devices` is a tool used to simulate a number of devices that reply to requests with a message of configurable size. Each simulated device in the Cometa Client Benchmark is able to:

* Connect to a Cometa server
* Send a heartbeat to maintain the connection
* Reply to a request with a message of a certain size

Command line parameters:

| PARAMETER        | DESCRIPTION                                                 |
 ------------------|----------------------------------|-------------------------------------
| `server_name`  | Cometa server IP address    
| `server_port`   | Cometa server port             
| `app_id`   | Cometa application id            
| `app_secret`   | Cometa application secret            
| `heartbeat`   | heartbeat in seconds (default is 60)            
| `device_id`   | numeric start device ID for the first device           
| `response_size`   | size in bytes of response message            
| `use_ssl `   | use SSL for server communication           
 
####Load Generator

For the benchmarks performed a load generator has been written for Node.js and to use WebSocket endpoints for devices. The load generator sends a generic `ping` message to a random device in the specified range using JavaScript timers to send the next message to the same device after a random period of time between 1 and 3 seconds.

While the shape of the traffic varies on network conditions, the server is constantly handling a large number of parallel requests from the load generator. 
 
#### Latency
For the purposes of this benchmarks, latency is defined as the time needed for a message to propagate from the Cometa server to the device and back. The latency is the difference between the time of arrival of a message to Cometa and the time the response is received from the device. 

Using this definition is possible to estimate the added latency for a message/reply RPC invocation to go through the Cometa server. It is also possible to understand how this added latency depends on the number of connected devices and the message rate. 

#### Latency Statistics
The Cometa server collects traffic statistics continuously and it is obtained by a HTTP GET to the `/application/{app_d}` resource as specified in the [Cometa API](http://www.cometa.io/cometa-api.html#application).

Example:
```
$ curl -H 'Authorization: OAuth 8ad3773142a6692b25b7' http://api.cometa.io/application/946604ed1d971eca2879

{
  "app_id":"946604ed1d971eca2879",
  "app_name":"cometatest",
  "stats": {
    "num_devices":"10000",
    "messages":"8896837",
    "errors":"0",
    "bytes_up":"302578104",
    "bytes_down":"286847744",
    "latency":"8",
    "latency_min":"0",
    "latency_max":"12",
    "std_deviation":"1.000",
    "websockets":"500"
  }
}

```
The latency measurements and the latency standard deviation are in milliseconds.  The latency is intended to be the arithmetic mean over the sampling period across all connected devices. Statistics has been collected every 15 seconds for the duration of all benchmarking sessions.

Benchmarking Setup
-------------
The test was performed using 18 EC2 instances on Amazon Web Services. One instance running the Cometa server and the others running the Cometa Client Benchmarking software. 

The instance running the Cometa server was a c4.4xlarge (62 ECUs, 16 vCPUs, 2.9 GHz, Intel Xeon E5-2666v3, 30 GiB memory, EBS) while the instances running the Client Benchmarking software were m3.large (2 vCPUs,  7,5 GiB memory, EBS). 

The machines were all running Linux Ubuntu 64-bit with kernel 3.8.0-19-generic. They were also all in the same AWS availability zone and in the same VPN, making the network delay negligible.

Benchmarking Results
------------
For each use case the following information is provided:

* Number of devices
* Device response size in bytes
*  Total number of messages received by the devices
* Messages received per second by each device (average)
*  Network upstream throughput in kB/sec (from the device to the Cometa server)
*  Network downstream throughput in kB/sec (from the Cometa server to the device)
* Average server latency in msec
* Minimum server latency in msec
* Maximum server latency in msec
* Latency standard deviation in msec
* CPU load average 

Each use case is the result of a 5 minute run at sustained load, that is after the time for all simulated devices to connect and the WebSockets opened. Each machine running the Cometa Client Benchmark has been used to simulate up to 60,000 devices and depending on the use case,  from 2 and up to 17 servers have been used to run the Cometa Client Benchmark at the same time, in addition to the larger server running the Cometa server.

The load generator application was running on a separate machine in the same availability zone, and it was configured to simulate 2,000 or 4,000 applications using a WebSocket each to connect to the Cometa server, resulting in hundreds of messages per second, and up to more than one thousand, sustained for the duration of each use case.

----------

### Case 1: 100,000 connected devices

#### Case 1.1:  response size 32 bytes
Number of devices: 100,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 287,723   | units    |
| Message rate     | 969       | msgs/sec |
| Throughput Up    | 31        | kB/sec   |
| Throughput Down  | 30        | kB/sec   |
| Avg. Latency     | 7         | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 34        | msec     |
| Std. Deviation   | 6         | msec     |
| CPU load average | 0.05      | %        |

#### Case 1.2:  response size 256 bytes
Number of devices: 100,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 313,379   | units    |
| Message rate     | 1,044     | msgs/sec |
| Throughput Up    | 263       | kB/sec   |
| Throughput Down  | 32        | kB/sec   |
| Avg. Latency     | 7         | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 40        | msec     |
| Std. Deviation   | 6         | msec     |
| CPU load average | 0.08      | %        |

#### Case 1.3:  response size 1,024 bytes
Number of devices: 100,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 303,018   | units    |
| Message rate     | 1,010     | msgs/sec |
| Throughput Up    | 1,012     | kB/sec   |
| Throughput Down  | 31        | kB/sec   |
| Avg. Latency     | 6         | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 44        | msec     |
| Std. Deviation   | 5         | msec     |
| CPU load average | 0.07      | %        |


----------
### Case 2: 500,000 connected devices

#### Case 2.1:  response size 32 bytes
Number of devices: 500,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 234,735   | units    |
| Message rate     | 782       | msgs/sec |
| Throughput Up    | 25        | kB/sec   |
| Throughput Down  | 24        | kB/sec   |
| Avg. Latency     | 10        | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 82        | msec     |
| Std. Deviation   | 9         | msec     |
| CPU load average | 0.1       | %        |

#### Case 2.2:  response size 256 bytes
Number of devices: 500,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 237,032   | units    |
| Message rate     | 790       | msgs/sec |
| Throughput Up    | 119       | kB/sec   |
| Throughput Down  | 24        | kB/sec   |
| Avg. Latency     | 9         | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 117       | msec     |
| Std. Deviation   | 9         | msec     |
| CPU load average | 0.11      | %        |

#### Case 2.3:  response size 1,024 bytes
Number of devices: 500,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 185,344   | units    |
| Message rate     | 617       | msgs/sec |
| Throughput Up    | 619       | kB/sec   |
| Throughput Down  | 19        | kB/sec   |
| Avg. Latency     | 6         | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 40        | msec     |
| Std. Deviation   | 6         | msec     |
| CPU load average | 0.11      | %        |

#### Case 2.4:  response size 1,024 bytes
Number of devices: 500,000
Number of opened WebSockets:  4,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 474,659   | units    |
| Message rate     | 1,582     | msgs/sec |
| Throughput Up    | 1,585     | kB/sec   |
| Throughput Down  | 49        | kB/sec   |
| Avg. Latency     | 12        | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 136       | msec     |
| Std. Deviation   | 23        | msec     |
| CPU load average | 0.22      | %        | 


----------
### Case 3: 1,000,000 connected devices

#### Case 3.1:  response size 32 bytes
Number of devices: 1,000,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 217,145   | units    |
| Message rate     | 723       | msgs/sec |
| Throughput Up    | 24        | kB/sec   |
| Throughput Down  | 22        | kB/sec   |
| Avg. Latency     | 27        | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 352       | msec     |
| Std. Deviation   | 20        | msec     |
| CPU load average | 0.2       | %        |

#### Case 3.2:  response size 256 bytes
Number of devices: 1,000,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 221,476   | units    |
| Message rate     | 738       | msgs/sec |
| Throughput Up    | 186       | kB/sec   |
| Throughput Down  | 23        | kB/sec   |
| Avg. Latency     | 22        | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 95        | msec     |
| Std. Deviation   | 15        | msec     |
| CPU load average | 0.27      | %        |

#### Case 3.3:  response size 1,024 bytes
Number of devices: 1,000,000
Number of opened WebSockets:  2,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 206,055   | units    |
| Message rate     | 686       | msgs/sec |
| Throughput Up    | 688       | kB/sec   |
| Throughput Down  | 21        | kB/sec   |
| Avg. Latency     | 30        | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 332       | msec     |
| Std. Deviation   | 26        | msec     |
| CPU load average | 0.25      | %        |

#### Case 3.4:  response size 1,024 bytes
Number of devices: 1,000,000
Number of opened WebSockets:  4,000

| PARAMETER        | VALUE     | UNIT     |
 ------------------|-----------|----------|
| Total Messages   | 438,805   | units    |
| Message rate     | 1,462     | msgs/sec |
| Throughput Up    | 1,465     | kB/sec   |
| Throughput Down  | 45        | kB/sec   |
| Avg. Latency     | 32        | msec     |
| Min. Latency     | 0         | msec     |
| Max. Latency     | 1373      | msec     |
| Std. Deviation   | 22        | msec     |
| CPU load average | 0.45      | %        |


------
&copy; 2015 Copyright,  Visible Energy Inc.
cometa@visiblenergy.com


