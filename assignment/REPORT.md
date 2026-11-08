# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2158**
## 1. Analysis: the loop

In C1 I tested what happens when a tunnelled packet enters a fowarding loop without a hop limit. I started the network using python3 topo.py --id 2158 and inspected MyIngress.myTunnel_exact table on switch s2. The entry for tunnel destination ID 3 handle 0x2 and originally forwarded packets through port 3 to h3. Ichanged this entry to forward packets through port 1 back towards s1 using (sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1") since s1 already forwards tunnel destination 3 towards s2 this created a forwarding loop.

Then I sent packet (h1 python3 send.py 10.58.0.3 "loop" --tunnel 3) The packet processing counters (s1,s2; first count: 328,327 after 5 seconds: both are 1509). The counters continued increasing, meaning that the tunnelled packet kept circulating between switches.

The reason is that the original myTunnel_forward action only assigns an output port. It does not decrement the IPv4 TTL or enforce a separate tunnel hop limit.For ordinary IPv4 forwarding, the ipv4_forward action decreases the TTL by one. But the supplied P4 program does not check whether the TTL has reached zero before forwarding.The reason is that the original myTunnel_forward action only assigns an output port. It does not decrement the IPv4 TTL or enforce a separate tunnel hop limit.

For ordinary IPv4 forwarding, the ipv4_forward action decreases the TTL by one but the supplied P4 program does not check whether the TTL has reached zero before forwarding. Since TTL is an 8-bit field, decrementing zero can wrap the value to 255. TTL decrementing alone does not reliably prevent IPv4 forwarding loops in this implementation. An explicit TTL expiration check is also required.

Evidence: assignment/evidence/loop-before.txt 

## 2. Design: the hop limit

I designed a hop limit to prevent tunnelled packets from circulating indefinitely. The new field is called hop_limit, unsigned 8-bit field placed after dst_id in custom tunnel header. The updated header consicts of proto_id (i16bits), dst_id(16bits) and hop_limit(8bits). The original tunner header was 4 bytes while the new one is 5 bytes. It is placed after the dst_id so that the positions would remain unchanged and allows an older switch to extract the original tunnel fields although it does not guanrantee complete packet compatibility. I chose an 8-bit field because one additional byte provides values from 0 to 255 with relatively low packet overhead.

The starting hop limit is 8, configured as the default value in send.py. I selected this value because the normal topology requires only two switch traversals, leaving enough allowance for additional hops while limiting loops. The forwarding logic follows these rules: If hop_limit is 0 or 1, drop the packet. Otherwise, decrease hop_limit by one. Apply myTunnel_exact to select the outgoing port.

For normal delivery, the hop limit decreases from 8 to 7 at s1 and to 6 at s2. In a loop, the packet is forwarded seven times and dropped on its eighth switch visit.The parser, deparser, sender, and receiver must use the same header layout to avoid incorrect packet interpretation.

## 3. Implementation

tunnel.p4
I added bit<8> hop_limit; to the myTunnel_t header. The parser extracts the updated header, while the deparser emits it with the modified hop-limit value. In the ingress pipeline, I added a condition to drop packets when hop_limit <= 1. Otherwise, the hop limit is decremented before applying myTunnel_exact. Ordinary IPv4 packets continue using ipv4_lpm

send.py
I updated the Scapy MyTunnel definition by adding: ByteField("hop_limit", 8) Also introduced the optional --hops argument. Its default value is 8, and the program validates that the supplied value is between 0 and 255. For example:
h1 python3 send.py 10.58.0.3 "delivery h2" --tunnel 2 --hops 8
This allows different starting values to be tested without changing the P4 program.

receive.py
Added the same 8-bit field to the receiver's Scapy tunnel-header definition. Also updated the packet display function to print the remaining hop limit alongside the tunnel destination, IP addresses, TTL, and payload. This makes it possible to verify how many forwarding decrements occurred.I preserved the original completed lab program as tunnel_v1.p4 so it could be used in the mixed-rollout experiment

## 4. Evidence: it works

### 4.1 Tunnel delivery
tested the updated implementation by sending packets to both tunnel destinations. For h2, used: h1 python3 send.py 10.58.0.3 "delivery h2" --tunnel 2 --hops 8

The receiving host displayed: Received (tunnelled, dst_id=2, hop_limit=6) from 10.58.0.1 to 10.58.0.3, TTL 64: 'delivery h2'

For h3, used: h1 python3 send.py 10.58.0.3 "delivery h3" --tunnel 3 --hops 8

The receiving host displayed:Received (tunnelled, dst_id=3, hop_limit=6) from 10.58.0.1 to 10.58.0.3, TTL 64: 'delivery h3'

Both packets arrived at their intended tunnel exits with a remaining hop limit of 6. This matches the expected decrement from 8 to 7 at s1 and from 7 to 6 at s2.
Evidence: assignment/evidence/delivery.txt

### 4.2 Ordinary IPv4 forwarding
Tested ordinary IPv4 forwarding using:

h1 ping -c 2 h3

Both ping packets succeeded with 0% packet loss and TTL 62. This confirms that the new tunnel hop-limit mechanism did not break ordinary IPv4 forwarding in the tested topology.
Evidence: assignment/evidence/delivery.txt

### 4.3 Tunnel loop after implementing the hop limit
I repeated the original forwarding-loop experiment by modifying s2's tunnel destination 3 entry to use port 1. Then sent:

h1 python3 send.py 10.58.0.3 "loop after" --tunnel 3 --hops 8

The counts remained unchanged, showing that the loop had stopped. The switch logs confirmed that `hop_limit <= 1` became true and the packet was dropped. The packet visited s1 four times and s2 four times. It was forwarded seven times and dropped on the eighth visit, matching the design

Evidence: assignment/evidence/loop-after.txt 


## 5. Evaluation

### a)The mixed rollout
**Prediction**
I predicted that the old switch s2 would still recognise proto_id and dst_id because their positions remained unchanged. However, it would not understand the additional hop_limit byte, potentially causing packet-format errors. I also expected inconsistent loop protection because only s1 would enforce the hop limit.

**Actual result**
 I started the network python3 topo.py --id 2158 --s2-program tunnel_v1.p4. Both programs compiled successfully. I sent a packet with dst_id=2 and starting hop limit 8. The s2 logs showed a successful table hit and forwarding through port 2, but receive.py at h2 did not display the packet. The tcpdump capture confirmed that the packet reached h2. The captured bytes included: 08 00 00 02 07 45 00 00 21

The old switch parsed only the original four-byte tunnel header, incorrectly interpreting the remaining hop-limit byte (0x07) as part of the IPv4 header. The captured IPv4 protocol byte changed from 0x11 (UDP) to 0x0d, demonstrating packet corruption. As a result, the receiving application could not decode the intended UDP message.

**Mixed-version loop**
I redirected tunnel destination 3 on s2 back to s1 and repeated the loop experiment. The counts remained unchanged. The s1 logs showed that the packet was dropped on its eighth visit to s1 when hop_limit <= 1 became true because only s1 decremented the hop limit, the mixed-version loop required more switch traversals before termination than the fully upgraded network.

Evidence: assignment/evidence/mixed.txt

### b)Safe rollout strategy
I considered two methods for safely deploying the new tunnel-header format.
**Option 1: New EtherType**
Assign a different EtherType to the updated five-byte tunnel header. This allows upgraded switches to distinguish new packets from the old 0x1212 format. However, older switches would not recognise the new EtherType, so compatibility gateways would be needed during the transition.

**Option 2: Version Field**
Add a version field so upgraded switches can identify and parse different tunnel-header formats. This supports future upgrades, but existing old switches cannot automatically understand the version field without an update or gateway.

My preferred approach is to use a new EtherType with compatibility gateways. I would upgrade the network gradually, starting with gateway support and testing before updating other switches. This prevents old switches from silently misinterpreting new headers

### c) Data-plane vs control-plane loop protection
The data-plane hop limit protects individual packets during forwarding. It stops packets from looping indefinitely, even when forwarding tables contain incorrect entries. However, it does not prevent those incorrect rules from being installed.

Week 3's control-plane check_paths() checks network paths before deployment. It can identify configuration problems such as loops or missing routes, but it may not detect unexpected runtime changes or differences between planned and installed rules.

Both are necessary because control-plane validation helps prevent loops, while the data-plane hop limit limits their impact if they still occur.

### d) Hardware implementation cost
The hop-limit mechanism adds one byte per tunnelled packet, increasing the tunnel header from four to five bytes. Each switch must extract the field, compare it against the expiration threshold, and decrement it before forwarding. These operations require some parser and pipeline resources. The operations are simple and may run at line rate on suitable programmable hardware. However, actual performance depends on hardware limitations. Deployment also requires compatible switch and host software. Hardware resource usage, throughput, and packet loss should be tested before production use.

## AI use

I used ChatGPT to understand the P4 lab instructions, interpret BMv2 parser and ingress logs, and obtain guidance on implementing the tunnel hop-limit field in P4 and Scapy. ChatGPT also assisted with debugging explanations, interpreting packet-processing counts, analysing the mixed-version tcpdump capture.

I ran the commands and experiments in my own GitHub Codespace and checked the results against my actual switch logs, packet captures, and receiver outputs.The submitted implementation and conclusions are based on the experiments recorded in the evidence files. 

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | Tunnel loop before implementing the hop limit |
| `evidence/delivery.txt` | Successful tunnel delivery and ordinary IPv4 forwarding |
| `evidence/loop-after.txt` | Automatic termination of the loop with hop-limit protection |
| `evidence/mixed.txt` |  Mixed-version packet corruption and loop termination |
