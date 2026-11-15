# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2380**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop
- The tunnel loop was created by re-pointing tunnel exit 3 on s2 which supposely points to h3 to sending packets back to s1 via port 1. With s1 already sending exit 3 to s2, the two switches will only pass the packet back and forth each other. The command adjusted was `sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward <handle> 1"`, which the handle is 2 after running `sh ./p4cli s2 table_dump MyIngress.myTunnel_exact` where handle 2 corresponds to tunnel dst_id 3.

- Initially, both switches had zero matching log entries. After about five seconds the counts increased to 105 packets and 103 packets for s1 and s2 respectively.

- The ipv4_forward decrements the IPv4 TTL based on `hdr.ipv4.ttl = hdr.ipv4.ttl - 1;`. However, when the ipv4_forward action is called, the TTL decreases by one but the action does not explicitly check whether the TTL has reached zero or drop the packet when it does.


## 2. Design: the hop limit
- The field size was set to 8 bits to repressent values from 0 to 255 which adds approximately only one byte of overhead, and it is placed after dst_id and at the end of the header to make it easier for the header to extend without changing the meaning of the existing fields and how its read, as such;

```
header myTunnel_t {
    bit<16> proto_id;
    bit<16> dst_id;
    bit<8> hop_limit;
}
```
- The starting value of the hop limit that send.py uses was set to 8 to allow the packets to travel across several switches while keeping it at a small limit to prevent them from looping excessively.  

- In the myTunnel_forward action, the hop limit is decreased by one everytime each switch forwards a tunnel packet. As the hop limit reaches 0, the switch drops the packet using the mark_to_drop().

```
action myTunnel_forward(egressSpec_t port) {
        if (hdr.myTunnel.hop_limit > 0) {
            hdr.myTunnel.hop_limit = hdr.myTunnel.hop_limit - 1;
            standard_metadata.egress_spec = port;
        } else {
            mark_to_drop(standard_metadata);
        }
```

-  Other than the header and ingress in tunnel.p4, the send.py and receive.py was also modified to recognise the new field added.


## 3. Implementation
**tunnel.p4**
- The header myTunnel_t now includes an 8 bit hop_limit and the myTunnel_forward action in the ingress pipeline was modified to check and decrement the hop_limit before forwarding a tunnel packet, as mentioned in Section 2.

```
 header myTunnel_t {
    bit<16> proto_id;
    bit<16> dst_id;
    bit<8> hop_limit;
}
```
```
action myTunnel_forward(egressSpec_t port) {
        if (hdr.myTunnel.hop_limit > 0) {
            hdr.myTunnel.hop_limit = hdr.myTunnel.hop_limit - 1;
            standard_metadata.egress_spec = port;
        } else {
            mark_to_drop(standard_metadata);
        }
```

**send.py**
- A new hop_limit field was added and updated in the MyTunnel class, and the default value was set to 8.

- Additionally, the starting value of hop_limit was also made configurable using --hops
```
    ap.add_argument("--hops", type=int, default=8,
                    help="starting tunnel hop limit (0-255)")
```
**receiver.py**
- The receiver also follows the same header structure as send.py by including the hop_limit field and its output was also modified to display the remaining hop limit for inspection

```
if MyTunnel in pkt:
     how = (
          f"tunnelled, dst_id={pkt[MyTunnel].dst_id}, "
          f"hop_limit={pkt[MyTunnel].hop_limit}"
     )
else:
     how = "plain IPv4"
```


## 4. Evidence: it works
- When `h1 python3 send.py 10.NN.0.2 "test h2" --tunnel 2 --hops 8` is run
```
###[ MyTunnel ]### 
     proto_id  = 2048
     dst_id    = 2
     hop_limit = 8
```
```
Listening on h2-eth0...
Received (tunnelled, dst_id=2, hop_limit=6) from 10.80.0.1 to 10.80.0.2, TTL 64: 'test h2'
Received (tunnelled, dst_id=2, hop_limit=6) from 10.80.0.1 to 10.80.0.2, TTL 64: 'test h2'
```
- When `h1 python3 send.py 10.NN.0.3 "test h3" --tunnel 3 --hops 8` is run
```
###[ MyTunnel ]### 
     proto_id  = 2048
     dst_id    = 3
     hop_limit = 8
```
```
== h3 logs 
Listening on h3-eth0...
Received (tunnelled, dst_id=3, hop_limit=6) from 10.80.0.1 to 10.80.0.3, TTL 64: 'test h3'
Received (tunnelled, dst_id=3, hop_limit=6) from 10.80.0.1 to 10.80.0.3, TTL 64: 'test h3'
```
- When `h1 ping -c 2 h3` is run
```
PING 10.80.0.3 (10.80.0.3) 56(84) bytes of data.
64 bytes from 10.80.0.3: icmp_seq=1 ttl=62 time=9.53 ms
64 bytes from 10.80.0.3: icmp_seq=2 ttl=62 time=9.51 ms

--- 10.80.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 9.514/9.520/9.527/0.006 ms
```

- After conducting the same loop as in Section 1 with `sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`, the packet counts remained unchanged, proving that the packets stopped circulating between switches instead of continuing indefinitely
```
logs/s1.txt:5
logs/s2.txt:4
== after 5 secs
logs/s1.txt:5
logs/s2.txt:4
```


## 5. Evaluation
**a) Prediction vs. Actual Results**
- I predicted that both would still work as usual, but may misalign their readings since s1 running on the new program has the proto_id, dst_id, and hop_limit as its headers, while s2 running on the old program only has proto_id and dst_id. Thus, s1's hop_limit may be read as a new packet's proto_id by s2.

- The output 
```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on h2-eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes

0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

```s1
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Processing packet received on port 1
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser 'parser': start
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser 'parser' entering state 'start'
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Extracting header 'ethernet'
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser state 'start': key is 1212
[21:40:15.915] [bmv2] [T] [thread 88659] [1.0] [cxt 0] Bytes parsed: 14
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser 'parser' entering state 'parse_myTunnel'
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Extracting header 'myTunnel'
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser state 'parse_myTunnel': key is 0800
[21:40:15.915] [bmv2] [T] [thread 88659] [1.0] [cxt 0] Bytes parsed: 19
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser 'parser' entering state 'parse_ipv4'
[21:40:15.915] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Extracting header 'ipv4'
[21:40:15.916] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser state 'parse_ipv4' has no switch, going to default next state
[21:40:15.916] [bmv2] [T] [thread 88659] [1.0] [cxt 0] Bytes parsed: 39
[21:40:15.916] [bmv2] [D] [thread 88659] [1.0] [cxt 0] Parser 'parser': end
```

```s2
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Processing packet received on port 1
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser 'parser': start
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser 'parser' entering state 'start'
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Extracting header 'ethernet'
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser state 'start': key is 1212
[21:40:15.918] [bmv2] [T] [thread 88679] [1.0] [cxt 0] Bytes parsed: 14
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser 'parser' entering state 'parse_myTunnel'
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Extracting header 'myTunnel'
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser state 'parse_myTunnel': key is 0800
[21:40:15.918] [bmv2] [T] [thread 88679] [1.0] [cxt 0] Bytes parsed: 18
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser 'parser' entering state 'parse_ipv4'
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Extracting header 'ipv4'
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser state 'parse_ipv4' has no switch, going to default next state
[21:40:15.918] [bmv2] [T] [thread 88679] [1.0] [cxt 0] Bytes parsed: 38
[21:40:15.918] [bmv2] [D] [thread 88679] [1.0] [cxt 0] Parser 'parser': end
```
**b) Safe way to roll out a header change**
Option 1: Introduce a new Ethertype
A new EtherType can identify the updated tunnel format. Switches can distinguish between old and new packet formats based on the Ethernet header.
- Old switches continue handling the original EtherType.
- Updated switches are configured to recognise the new EtherType and parse the new header.
- This reduces the risk of an old switch interpreting a new header incorrectly.
However, old switches must still know how to forward the new EtherType. Otherwise, packets using the new format may be dropped or mishandled.

Option 2: Use a version field and backward-compatible parsing
A version field can identify which tunnel header format is being used.
- Updated switches can recognise both the old and new formats.
- They can parse packets according to the version number.
- The network can gradually migrate traffic to the new format.
However, this requires updated switches to support both formats during the transition. Old switches that do not understand the version field cannot automatically interpret the new format.

**c) Data Plane and Control Plane Loop Protection**
- In my opinion, both are as important since the check_paths() gives a more preventive approach as it prevents configuration mistakes while the hop limit goes for a detective approach and provides an additional layer of protection when packets are being forwarded.

- The hop limit cannot identify whether a forwarding path is logically correct, but it can limit how long a packet can continue travelling. While check_paths() is able to identify a loop in a configured topology, but it cannot necessarily detect unexpected runtime forwarding behavior or incorrect switch state.

**d) Hardware Cost**
- Header overhead where one byte is added to every tunnel packet, 
- Processing overhead where the switch needs to read the hop limit, compare to zero, and decrement everytime when forwarding. 
- Hardware resources where the parser must extract an additional field, while the ingress pipeline conducts the comparison, decrement, and drop operations.


## AI use
ChatGPT was used for the following:
- Explained how to read the header, parser, ingress, egress, deparser, and what each stage does; difference between using the tunnel or ipv4 forward actions; and how to read the logs
- Modifying tunnel.p4, send.py and adding --hops, receive.py
- Understanding the mixed rollout and its expected output based on s1 and s2 logs, and answering questions based on the mixed rollout

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
