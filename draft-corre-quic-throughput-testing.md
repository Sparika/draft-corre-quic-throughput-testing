---
title: "Framework for QUIC Throughput Testing"
#abbrev: "TODO - Abbreviation"
docname: draft-corre-quic-throughput-testing-latest
category: info

ipr: trust200902
#area: TODO
workgroup: Network Working Group
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Kevin Corre
    organization: EXFO
    email: kevin.corre@exfo.com

normative:
  RFC6349:
  RFC9002:
  RFC1191:
  RFC793:
  RFC6582:
  RFC9001:
  RFC4821:
  RFC9000:
  RFC8899:

informative:
  marx2020same:
    title: "Same standards, different decisions: A study of quic and http/3 implementation diversity"
    author:
      -
        initials: R.
        surname: Marx
        fullname: Marx Robin
      -
        initials: J.
        surname: Herbots
        fullname: Joris Herbots
      -
        initials: W.
        surname: Lamotte
        fullname: Wim Lamotte
      -
        initials: P.
        surname: Quax
        fullname: Peter Quax
    date: 2020
    refcontent:
      "Proceedings of the Workshop on the Evolution, Performance, and Interoperability of QUIC"
    DOI: 10.1145/3405796.3405828


--- abstract

This document adapts the {{RFC6349}} Framework for TCP Throughput Testing to the {{RFC9000}} QUIC protocol.
The adapted framework describes a practical methodology for measuring end-to-end QUIC Throughput in a managed IP network.
The goal of the methodology is to provide a better indication in regard to user experience.
In this framework, QUIC, UDP, and IP parameters are specified to optimize QUIC Throughput.


--- middle

# Introduction

This document adapts the {{RFC6349}} Framework for TCP Throughput Testing to the QUIC protocol {{RFC9000}}.
RFC6349 defines a methodology for testing sustained TCP Layer performance.
Like TCP {{RFC793}}, QUIC is a connection-oriented transport protocol.
However, there are multiple differences between both protocols and some of these differences impact the throughput testing methodology.

For easier reference, this document follows the same organization as {{RFC6349}} and each section describes changes relevant to the equivalent {{RFC6349}} section.
The scope and goals section is omitted as the objective of this document is precisely to remain within the same scope and goals of {{RFC6349}}.
{{methodology}} presents changes to the individual steps of the {{RFC6349}} methodology.
{{metrics}} describes the three main metrics used to measure QUIC Throughput.
{{conducting}} covers conducting QUIC Throughput testing.
In particular, the section presents QUIC streams usage in the context of QUIC Throughput testing and discusses possible results interpretation.
Finally, {{security}} presents additional security considerations.

## Impacting changes between TCP and QUIC {#changes}

This methodology proposes QUIC Throughput performance testing focusing on:

- Bottleneck Bandwidth (BB)
- Round-Trip Time (RTT)
- Send and Receive Socket Buffers
- Available Control-Flow Credits and QUIC CWND
- Path Maximum Transmission Unit (MTU)

There are multiple changes between TCP and QUIC impacting the throughput testing methodology.
Firstly, the multiple QUIC headers and frames result in overheads variable in size which in turn impact the computation for the maximum achievable QUIC throughput, in particular the Transfert Time Ration metric presented in {{transferttimeratio}}.
Secondly, QUIC provides streams that can be used for throughput testing but which may also result in variable overheads.

TODO:

Thirdly, QUIC provides congestion control genericity and receiver-controlled control flow credits instead of the TCP sliding receiver window.
While QUIC Loss Detection and Congestion Control {{RFC9002}} exmplifies with a congestion controller similar to TCP NewReno~{{RFC6582}},
the signals QUIC provides for congestion control are generic and are designed to support different sender-side algorithms.
In this document, the achievable QUIC Throughput is the amount of data per unit of time that QUIC transports when in the QUIC Equilibrium state.
Derived from Round-Trip Time (RTT) and network Bottleneck Bandwidth (BB), the Bandwidth-Delay Product (BDP) determines the Send and Received Socket buffer sizes required to achieve the maximum QUIC Throughput.
Throughout this document, "maximum achievable throughput" refers to the theoretical achievable throughput when QUIC is in the Equilibrium state.
In this document, we assume that QUIC uses a transmitting side congestion controller with a congestion window (QUIC CWND) and congestion controller states similar to TCP NewReno.

~~~
                 New path or      +------------+
            persistent congestion |   Slow     |
        (O)---------------------->|   Start    |
                                  +------------+
                                        |
                                Loss or |
                        ECN-CE increase |
                                        v
 +------------+     Loss or       +------------+
 | Congestion |  ECN-CE increase  |  Recovery  |
 | Avoidance  |------------------>|   Period   |
 +------------+                   +------------+
           ^                            |
           |                            |
           +----------------------------+
              Acknowledgment of packet
                sent during recovery
~~~
{: #fig-cc-fsm title="QUIC NewReno Congestion Control States and Transitions"}

On the receiving side, QUIC does not use a sliding receiver window and instead uses a control flow credits mechanism to inform the transmitting end on the maximum total amount of data the receiver is willing to receive for each stream and connection.
The amount of available control flow credits does not directly control the QUIC Throughput but insufficient credits may lead to a stream or a whole connection being rate-limited.
This is discussed in {{cfc}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Methodology {#methodology}

Overall, the testing methodology remains the same as the one described in {{RFC6349}} with a few minor differences.
The following represents the sequential order of steps for this testing methodology:

1. Identify the Path MTU.
As QUIC relies on UDP, Packetization Layer Path MTU Discovery for Datagram Transports (Datagram PMTUD) {{RFC8899}} SHOULD be conducted instead of Packetization Layer Path MTU Discovery (PLPMTUD).
It is important to identify the path MTU so that the QUIC Throughput Test Device (TTD) is configured properly to avoid fragmentation.

2. Baseline Round-Trip Time and Bandwidth.
This step establishes the inherent, non-congested Round-Trip Time (RTT) and the Bottleneck Bandwidth (BB) of the end-to-end network path.
These measurements are used to provide estimates of the QUIC minimum congestion control credit and Send Socket Buffer sizes that SHOULD be used during subsequent test steps.

3. QUIC Connection Throughput Tests.
With baseline measurements of Round-Trip Time and Bottleneck Bandwidth, single- and multiple-QUIC-connection throughput tests SHOULD be conducted to baseline network performance.

These three (3) steps are detailed in {{mtu}} to {{throughput}}.

Regarding, the QUIC TTD, the same key characteristics and criteria SHOULD be considered than with the {{RFC6349}} TCP TTD.
The test host MAY be a standard computer or a dedicated communications test instrument and in both cases, it MUST be capable of emulating both a client and a server.
One major difference is that contrary to TCP, QUIC may not be provided by OSs usually providing transport protocol implementations and may instead be sourced by an application from a third-party library.
These different implementations may present "a large behavioural heterogeneity" {{marx2020same}} potentially impacting the QUIC throughput measurements.
Thus, in addition to the OS version (e.g. a specific LINUX OS kernel version), the QUIC implementation used by the test hosts MUST be considered, including used congestion control algorithms available and supported QUIC options.
The QUIC test implementation and hosts MUST be capable of generating and receiving stateful QUIC test traffic at the full BB of the NUT.

QUIC includes precise RTT statistics and MAY be directly used to gather RTT statistics if the QUIC implementation exposes them.
As QUIC packets are mostly encrypted, packet capture tools do not allow to measure QUIC RTT and retransmissions and SHOULD not be used for this methodology.

## Path MTU {#mtu}
QUIC implementations SHOULD use Datagram Path MTU Discovery techniques (Datagram PMTUD).

## Round-Trip Time (RTT) and Bottleneck Bandwidth (BB) {#rtt}
Before stateful QUIC testing can begin, it is important to determine the baseline RTT (i.e. non-congested inherent delay) and BB of the end-to-end network to be tested.
These measurements are used to calculate the BDP and to provide estimates for the congestion control credit and Send Socket Buffer sizes that SHOULD be used in subsequent test steps.

## Measuring RTT
In addition to the solutions proposed in {{RFC6349}} for measuring RTT, short QUIC sessions MAY be employed to baseline RTT values off-peak hours and outside of test sessions.
This solution requires that the QUIC implementation expose RTT measurements {{RFC9002}}.
latest_rtt SHOULD be used to sample baseline RTT using the minimum observed latest_RTT during off-peak hours and outside of test sessions.
instead of latest_rtt, min_rtt MAY be used to sample baseline RTT during off-peak hours and outside of test sessions.
smoothed_rtt MUST not be used to baseline RTT.

## Measuring BB
This section is unchanged.

## Measuring QUIC Throughput {#throughput}
This methodology specifically defines QUIC Throughput measurement techniques to verify maximum achievable QUIC performance in a managed business-class IP network.

With baseline measurements of RTT and BB from {{rtt}}, a series of single- and/or multiple-QUIC-connection throughput tests SHOULD be conducted.

Like with TCP, the number of trials and the choice between single or multiple QUIC connections will be based on the intention of the test.
In all circumstances, it is RECOMMENDED to run the tests in each direction independently first and then to run them in both directions simultaneously.
It is also RECOMMENDED to run the tests at different times of the day.

The metrics measured are the QUIC Transfer Time Ratio, the QUIC Efficiency Percentage, and the Buffer Delay Percentage.
These 3 metrics are defined in {{metrics}} and MUST be measured in each direction.

## Minimum QUIC Congestion Control Credit {#cfc}
As for {{RFC6349}} TCP throughput testing, the QUIC throughput test MUST ensure that the QUIC performance is never rate-limited.
Whereas TCP uses a sliding receiver window (TCP RWND), QUIC relies on congestion control credits that are not automatically sliding.
Instead, available control-flow credits are increased by sending MAX_DATA and MAX_STREAM_DATA frames.
The algorithm used to send these frames depends on the QUIC implementation or may be left for the application to control at each receiving side.

In addition to setting Send Socket Buffer size higher than the BDP, the QUIC TTD MUST ensure that connections and streams always have Control-Flow Credits (CFC) in excess of the BDP so that any QUIC stream or connection is never restricted below the BDP before MAX_DATA and MAX_STREAM_DATA frames have a chance to arrive.
If a QUIC connection or stream has CFC below the minimum required CFC, then there is a risk of the CFC reaching 0 before the limits can be increased.
If a QUIC stream has CFC at 0, then that stream is rate-limited and there is a risk of the QUIC Throughput to not be optimal.
Note that other streams may still manage to fill the BDP of the NUT.
If a QUIC connection has CFC at 0, then that connection is rate-limited and the QUIC Throughput cannot be optimal.
The QUIC TTD MUST report every events and events' duration where a QUIC stream or connection is rate-limited.

A QUIC implementation could implement a sliding receive window similar to TCP RWND by sending MAX_DATA and MAX_STREAM_DATA frames with each ACK frame.
In this case, the minimum CFC (i.e. initial data limit) is similar to the minimum TCP RWND and example numbers proposed in {{RFC6349}}.
However, such a solution imposes a heavy overhead and it is more likely that sending of MAX_DATA and MAX_STREAM_DATA frames are delayed by the QUIC implementation.
Thus, the minimum CFC should be set sufficiently large so that the QUIC implementation can send MAX_DATA and MAX_STREAM_DATA frames with new limits before control-flow credits are exhausted.

# QUIC Metrics {#metrics}

The proposed metrics for measuring QUIC Throughput remain the same as for measuring TCP Throughput with some minor differences.

## Transfert Time Ratio {#transferttimeratio}

The first metric is the QUIC Transfer Time Ratio, which is the ratio between the Actual QUIC Transfer Time versus the Ideal QUIC Transfer Time.
The Actual QUIC Transfer Time is simply the time it takes to transfer a block of data across QUIC connection(s).
The Ideal QUIC Transfer Time is the predicted time for which a block of data SHOULD transfer across QUIC connection(s), considering the BB of the NUT.

~~~
      QUIC          Actual QUIC Transfer Time
  Transfer Time =  ---------------------------
      Ratio          Ideal QUIC Transfer Time
~~~

The Ideal QUIC Transfer Time is derived from the Maximum Achievable QUIC Throughput, which is related to the BB and Layer 1/2/3/4 overheads associated with the network path.
The following sections provide derivations for the Maximum Achievable QUIC Throughput and example calculations for the QUIC Transfer Time Ratio.

### Maximum Achievable QUIC Throughput Calculation

This section provides formulas to calculate the Maximum Achievable QUIC Throughput, with examples for T3 (44.21 Mbps) and Ethernet.

On the contrary to TCP, the QUIC overhead is variable in size, in particular, due to variable-length fields and streams multiplexing.
Furthermore, QUIC is a versioned protocol and the overheads may change from version to version.
Other underlying protocol changes such as IP version may also increase overheads or make them variable in size.
The following calculations are based on IP version 4 with IP headers of 20 Bytes, UDP headers of 8 Bytes, QUIC version 1, and within an MTU of 1500 Bytes.

We are interested in estimating the maximal QUIC Throughput at equilibrium, so we first simplify the problem by considering an optimistic scenario with data exchanged over 1-RTT packets with only a single stream frame per packet.
For the content of the STREAM frame, we consider that the offset field is used but not the length field since there is only one frame per packet.

Here are the 1-RTT packet and STREAM frame headers as defined in {{RFC9000}}.

~~~
   1-RTT Packet {
     Header Form (1) = 0,
     Fixed Bit (1) = 1,
     Spin Bit (1),
     Reserved Bits (2),
     Key Phase (1),
     Packet Number Length (2),
     Destination Connection ID (0..160),
     Packet Number (8..32),
     Packet Payload (8..),
   }
~~~

~~~
   STREAM Frame {
     Type (i) = 0x08..0x0f,
     Stream ID (i),
     [Offset (i)],
     [Length (i)],
     Stream Data (..),
   }
~~~

Supposing all variable-length fields are encoded as the minimum size possible, the overhead would be 3 Bytes for QUIC 1-RTT packet and 2 Bytes for QUIC STREAM frame.
This gives us a minimal IP/UDP/QUIC overhead of 20 + 8 + 3 + 2 = 33 Bytes.
Note that alternatively, considering all variable length fields encoded as the largest size possible, the overhead would be 25 Bytes for QUIC 1-RTT packet and 17 Bytes for QUIC STREAM frame.
In this worst-case scenario, the overhead could mount up to 20 + 8 + 25 + 17 = 70 Bytes for one STREAM frame per packet.

To estimate the Maximum Achievable QUIC Throughput, we first need to determine the maximum quantity of Frames Per Second (FPS) permitted by the actual physical layer (Layer 1) speed.

For a T3 link, the calculation formula is:

~~~pseudocode
  FPS = T3 Physical Speed /
        ((MTU + PPP + Flags + CRC16) X 8)

  FPS = (44.21 Mbps /
        ((1500 Bytes + 4 Bytes
          + 2 Bytes + 2 Bytes) X 8 )))
  FPS = (44.21 Mbps / (1508 Bytes X 8))
  FPS = 44.21 Mbps / 12064 bits
  FPS = 3664
~~~

Then, to obtain the Maximum Achievable QUIC Throughput (Layer 4), we simply use:

~~~pseudocode
  (MTU - 33) in Bytes X 8 bits X max FPS
~~~

For a T3, the maximum QUIC Throughput =

~~~pseudocode
  1467 Bytes X 8 bits X 3664 FPS

  Max QUIC Throughput = 11736 bits X 3664 FPS
  Max QUIC Throughput = 43.0 Mbps
~~~

On Ethernet, the maximum achievable Layer 2 throughput is limited by the maximum Frames Per Second permitted by the IEEE802.3 standard.

The maximum FPS for 100-Mbps Ethernet is 8127, and the calculation formula is:

~~~pseudocode
  FPS = (100 Mbps / (1538 Bytes X 8 bits))
~~~

The maximum FPS for GigE is 81274, and the calculation formula is:

~~~pseudocode
  FPS = (1 Gbps / (1538 Bytes X 8 bits))
~~~

The maximum FPS for 10GigE is 812743, and the calculation formula is:

~~~pseudocode
  FPS = (10 Gbps / (1538 Bytes X 8 bits))
~~~

The 1538 Bytes equates to:

~~~pseudocode
  MTU + Ethernet + CRC32 + IFG + Preamble + SFD
   IFG = Inter-Frame Gap
   SFD = Start of Frame Delimiter
~~~

where MTU is 1500 Bytes, Ethernet is 14 Bytes, CRC32 is 4 Bytes, IFG is 12 Bytes, Preamble is 7 Bytes, and SFD is 1 Byte.

Then, to obtain the Maximum Achievable QUIC Throughput (Layer 4), we simply use:

~~~pseudocode
  (MTU - 33) in Bytes X 8 bits X max FPS
~~~

   For 100-Mbps Ethernet, the maximum QUIC Throughput =

~~~pseudocode
  1467 Bytes X 8 bits X 8127 FPS

  Max QUIC Throughput = 11736 bits X 8127 FPS
  Max QUIC Throughput = 95.4 Mbps
~~~


### QUIC Transfer Time and Transfer Time Ratio Calculation

The following table illustrates the Ideal QUIC Transfer Time of a single QUIC connection when its Send Socket Buffer size equals or exceeds the BDP and the sender is never control-flow limited.


| Link Speed (Mbps) | RTT (ms) | BDP (KBs) | Max. Ach. QUIC Thr. (Mbps) | Ideal QUIC Transfer Time (s)* |
|------------------:|---------:|----------:|---------------------------:|------------------------------:|
| 1.536             | 50.00    | 9.6       | 1.4                        | 571.0                         |
| 44.210            | 25.00    | 138.2     | 43.0                       | 18.6                          |
| 100.000           | 2.00     | 25.0      | 95.4                       | 8.4                           |
| 1,000.000         | 1.00     | 125.0     | 953.7                      | 0.8                           |
| 10,000.000        | 0.05     | 62.5      | 9,537.5                    | 0.1                           |
{: #100-mb-file-metrics-table title="Link Speed, RTT, BDP, Maximum QUIC Throughput, and Ideal QUIC Transfer Time for a 100-MB File"}

<aside markdown="block">
Note (*): Transfer times are rounded for simplicity.
</aside>



For a 100-MB file (100 X 8 = 800 Mbits), the Ideal QUIC Transfer Time is derived as follows:

~~~pseudocode
                            800 Mbits
    Ideal QUIC    = -------------------------
  Transfer Time         Maximum Achievable
                         QUIC Throughput
~~~

To illustrate the QUIC Transfer Time Ratio, an example would be the bulk transfer of 100 MB over 5 simultaneous QUIC connections  (each connection transferring 100 MB).
In this example, the Ethernet service provides a Committed Access Rate (CAR) of 500 Mbps.
Each connection may achieve different throughputs during a test, and the overall throughput rate is not always easy to determine (especially as the number of connections increases).

The Ideal QUIC Transfer Time would be ~8.4 seconds, but in this example, the Actual QUIC Transfer Time was 12 seconds.
The QUIC Transfer Time Ratio would then be 12/8.4 = 1.43, which indicates that the transfer across all connections took 1.43 times longer than the ideal.
Note that our estimation of the Maximum Achievable QUIC Throughput is overestimated due to the optimistic scenario initially described.
The actual QUIC Transfer Time is thus expected to be higher than the Ideal QUIC Transfer Time due to eventual network inefficiencies but also by overheads caused by the selected QUIC implementation as well as the number of concurrent streams being used for the test.

## QUIC Efficiency {#efficiency}

In {{RFC6349}}, the TCP efficiency metric represents the percentage of Bytes that were not retransmitted.
However, when packet loss occurs in QUIC, Bytes are not retransmitted per se, instead, new packets are generated.
As a result, a packet may contain retransmitted frames as well as new frames, and measuring the amount of retransmitted Bytes is not straightforward.

Instead, the QUIC efficiency MUST be measured by calculating the ratio between QUIC Payload Bytes and UDP Bytes.
QUIC Payload Bytes are the total number of payload Bytes sent or received over QUIC while UDP Bytes are the total number of bytes sent or received on the UDP socket, i.e. including QUIC protocol overheads.
This method allows to capture the impact of retransmission as retransmission of frames will impact the UDP Bytes without impacting the QUIC Bytes.
Retransmitted data will effectively lower the measured QUIC Efficiency.

~~~pseudocode
               QUIC payload Bytes
    QUIC =  -----------------------  X 100
  Efficiency         UDP Bytes
~~~


In comparison with {{RFC6349}}, the sender side QUIC Efficiency metric also measures the inherent overheads of QUIC caused by headers, frames, multiple streams, and implementation choices.
As the receiver side UDP Bytes also measures these overheads but does not measure retransmission overheads, the receiver side QUIC Efficiency MAY be used to normalize the sender side QUIC Efficiency.

~~~pseudocode
 Normalized Sender QUIC Efficiency
    QUIC =  --------------------------
 Efficiency  Receiver QUIC Efficiency
~~~

### QUIC Efficiency Percentage Calculation

As an example, considering a scenario where the MTU is 1500 bytes and the QUIC overhead is 50 bytes per packet on average. An application sending 1,000,000 Bytes over QUIC would emit approximately 1,000,000/1450 = 690 UDP packets of 1500 bytes resulting in a total of 690 x 1500 = 1,035,000 UDP Bytes.  The QUIC Efficiency Percentage would be calculated as:

~~~pseudocode
                        1,000,000
  QUIC Efficiency % =  -----------  X 100 = 96,6%
                        1,035,000
~~~

Considering a similar example with 1% of packet loss, QUIC would emit approximately 690 x 1.01 = 697 (round up) and the UDP Bytes emitted would be approximately  1,045,500 Bytes. The resulting QUIC Efficiency Percentage would thus be calculated as:

~~~pseudocode
                        1,000,000
  QUIC Efficiency % =  -----------  X 100 = 95,7%
                        1,045,500
~~~

On the receiver side, the measured UDP Bytes would be approximately 1,035,000 UDP Bytes in both scenarios since no loss overheads would be measured resulting in Receiver QUIC Efficiency = 96,6%.
Normalizing the previously measured QUIC Efficiency metrics (sender side) would thus result in Normalised QUIC Efficiency = 100% in the first example and Normalised QUIC Efficiency = 99,1%  in the example with 1% of packet loss.


## Buffer Delay
The third metric is the Buffer Delay Percentage, which represents the increase in RTT during a QUIC Throughput test versus the inherent or baseline RTT.
This metric remains identical to the one described in {{RFC6349}}.

~~~pseudocode
  Average RTT     Total RTTs during transfer
during transfer = ----------------------------
                  Transfer duration in seconds


                        Average RTT
                      during transfer
                       - Baseline RTT
  Buffer Delay % = --------------------- X 100
                        Baseline RTT
~~~

Note that while {{RFC9002}} specifies QUIC RTT measurements in the form of latest_rtt, smoothed_rtt, and min_rtt, the average RTT used in the Buffer Delay Percentage metric is derived from the total of all measured RTTs during the actual test at every second divided by the test duration in seconds.
RTT metrics smoothed_rtt and min_rtt described in {{RFC9002}} MUST NOT be used to compute the Average RTT during transfer.
RTT metric latest_rtt MAY be used for computing Average RTT during transfer by sampling it at every second.


# Conducting QUIC Throughput Tests {#conducting}
The methodology for QUIC Throughput testing follows the same considerations as described in {{RFC6349}}.

In the end, a QUIC Throughput Test Device (QUIC TTD) SHOULD generate a report with the calculated BDP and QUIC Throughput results.
The report SHOULD also include the results for the 3 metrics defined in Section 4.
As QUIC does not use a receive window, a QUIC Throughput test is constituted of multiple Send and Receive Socket Buffer experiments.
The report SHOULD include QUIC Throughput results for each Socket Buffer size tested.
The goal is to provide achievable versus actual QUIC Throughput results for the Socket Buffer size tested when no fragmentation occurs and to provide a clear relationship between these 3 metrics and user experience.
As an example, for the same Transfer Time Ratio, a better QUIC Efficiency could be obtained at the cost of higher Buffer Delays.

## Using Multiple QUIC Streams
In addition to opening multiple connections, QUIC allows data to be exchanged over fully multiplexed streams.
The primary goal of the methodology relates to ensuring the capacity of a managed network to deliver a particular SLA using QUIC.
As the NUT cannot observe the number of streams opened for a QUIC connection, we do not expect the NUT to apply specific policies based on the number of streams.
In the basic test scenario, A QUIC TTD MAY use two streams over a single connection where one stream is a test stream filling the BDP of the NUT and the other is a control stream used for controlling the test and exchanging test results.

Implementers of a QUIC Throughput test may however want to use multiple streams in parallel.
On one hand, using multiple streams over a single QUIC connection may result in a variable number of STREAM frames per packet, with the actual numbers depending on the number of streams, the amount of data per stream, and the QUIC implementation {{marx2020same}}.
This overhead variability may lower the QUIC Efficiency, increase the QUIC Transfer Time Ratio, and more generally reduce the reproducibility of the test with respect to the capacity of the NUT to provide a SLA.

On the other hand, using multiple streams over a single connection is a typical scenario for using QUIC and one of the selling-point of the protocol compared to TCP.
Unstable networks in particular those with a high packet loss are expected  benefit from stream multiplexing.
Implementing more complex scenarios in a QUIC Throughput test to better represent the use-cases of the NUT is possible.
A QUIC TTD MAY use multiple test streams to fill the BDP of the NUT.
In such case, the QUIC TTD SHOULD also run the test using a single test stream in order to facilitate root-cause analysis.
The specific parameters of a scenario using multiple test streams depends however on the use-case being considered.
For instance, emulating multiple HTTP/3 browsing sessions may involve a large number of short-lived small streams, while emulating real-time conversationnal streaming sessions may involve long-lived streams with large chunks of data.
Details for such scenarios and relevant QoE influence factors are out-of-scope of the proposed methodology.

## 1-RTT and 0-RTT

## Results Interpretation

For cases where the test results are not equal to the ideal values, some possible causes are as follows:

   - Network congestion causing packet loss, which may be inferred from a poor Normalised QUIC Efficiency % (i.e. higher Normalised QUIC Efficiency % = less packet loss).

   - Too many QUIC streams in parallel or an inefficient QUIC implementation with regards to protocol overheads may be inferred from a poor QUIC Efficiency
at both the sender and receiver sides (i.e. low QUIC Efficiency but high Normalised QUIC Efficiency).

   - Rate limiting by traffic policing may cause packet loss (i.e. low Normalised QUIC Efficiency).

   - Network congestion causing an increase in RTT, which may be inferred from the Buffer Delay Percentage (i.e. 0% = no increase in RTT over baseline).

   - Rate limiting by traffic shaping may also cause an increase in RTT.

   - Maximum UDP Buffer Space.  All operating systems have a global mechanism to limit the quantity of system memory to be used by UDP
     sockets.  On some systems, each socket is subject to a
     memory limit that is applied to the total memory used for input data, output data, and controls.  On other systems, there are separate limits for input and output buffer spaces per socket.
     Client/server IP hosts might be configured with Maximum UDP Buffer
     Space limits that are far too small for high-performance networks. These socket buffers MUST be large enough to hold a full BDP of UDP Bytes plus some overhead.

   - Path MTU.  The client/server IP host system SHOULD use the largest possible MTU for the path.  This may require enabling Path MTU
     Discovery {{RFC1191}} and {{RFC4821}}.  Since {{RFC1191}} is flawed, Path
     MTU Discovery is sometimes not enabled by default and may need to be explicitly enabled by the system administrator.  {{RFC4821}}
     describes a new, more robust algorithm for MTU discovery and ICMP
     black hole recovery.


# Security Considerations {#security}

Measuring QUIC network performance raises security concerns.
Metrics produced within this framework may create security issues.

Security considerations mentionned in {{RFC6349}} remain valid for QUIC Throughput testing.
In particular, as QUIC encrypts traffic over UDP, QUIC Throughput testing may appear as a denial-of-service attack.
Cooperation between the end-user customer and the network provider is thus required.

## 0-RTT attack
In practice, a QUIC Throughput test probably implements a control channel to control the test and one or more throughput channels to send data.
The control channel would serve to authenticate the TTD, request a test session through some throughput channels, and exchange test results.
As 0-RTT packets are vulnerable to replay attacks {{RFC9001}}, 0-RTT packets MUST not be used by a TTD client initiating a control channel to a TTD server.


# IANA Considerations

This document has no IANA actions.


--- back
