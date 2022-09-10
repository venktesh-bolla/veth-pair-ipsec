###Total LINUX NETNS CONFIG USING VETH PAIRS

# __________    <route_h1_gw1>   ____________________   <route_gw1_gw2>   ____________________    <route_gw2_h2>  ____________
#| 30.0.0.11|   <30.0.0.0/24>   |30.0.0.1   101.0.0.1|  <101.0.0.0/24>   |101.0.0.2   20.0.0.2|   <20.0.0.0/24>  |20.0.0.12   |
#|     veth1|<----------------->|veth1          veth2|<=================>|veth1          veth2|<---------------->|veth1       |
#|          |                   |                    |  <ipsec tunnel>   |                    |                  |            |
#|    x86   |                   |                    |                   |                    |                  |    x86     |
#|   HOST2  |                   |        x86         |                   |        x86         |                  |   HOST2    |
#|          |                   |     Linux-IPsec    |                   |    Linux-IPsec     |                  |            |
#|          |                   |       (setkey)     |                   |      (setkey)      |                  |            |
#|          |                   |                    |                   |                    |                  |            |
#|   H1NS   |                   |       GW1NS        |                   |       GW2NS        |                  |    H2NS    |
#|__________|                   |____________________|                   |____________________|                  |____________|
#

ip netns del h1ns
ip netns del gw1ns
ip netns del gw2ns
ip netns del h2ns

ip netns add h1ns
ip netns add gw1ns
ip netns add gw2ns
ip netns add h2ns

sysctl net.ipv4.ip_forward=1

ip link add veth1 netns h1ns  type veth peer name veth1 netns gw1ns
ip link add veth2 netns gw1ns type veth peer name veth1 netns gw2ns
ip link add veth2 netns gw2ns type veth peer name veth1 netns h2ns

ip netns exec h1ns  ifconfig veth1 up
ip netns exec h1ns  ifconfig veth1 30.0.0.11
ip netns exec h1ns  ip route add 20.0.0.0/24 via 30.0.0.1
ip netns exec h1ns  sysctl net.ipv4.ip_forward=1

ip netns exec gw1ns ifconfig veth1 up
ip netns exec gw1ns ifconfig veth2 up
ip netns exec gw1ns ifconfig veth1 30.0.0.1
ip netns exec gw1ns ifconfig veth2 101.0.0.1
ip netns exec gw1ns sysctl net.ipv4.ip_forward=1
ip netns exec gw1ns ip route add 20.0.0.0/24 via 101.0.0.2

ip netns exec gw2ns ifconfig veth1 up
ip netns exec gw2ns ifconfig veth2 up
ip netns exec gw2ns ifconfig veth1 101.0.0.2
ip netns exec gw2ns ifconfig veth2 20.0.0.2
ip netns exec gw2ns sysctl net.ipv4.ip_forward=1
ip netns exec gw2ns ip route add 30.0.0.0/24 via 101.0.0.1

ip netns exec h2ns  ifconfig veth1 up
ip netns exec h2ns  ifconfig veth1 20.0.0.12
ip netns exec h2ns  ip route add 30.0.0.0/24 via 20.0.0.2
ip netns exec h2ns  sysctl net.ipv4.ip_forward=1


ip netns exec gw1ns setkey -f setkey-gw1.conf
ip netns exec gw2ns setkey -f setkey-gw2.conf

#ip netns exec h1ns ping 20.0.0.12
#ip netns exec h2ns ping 13.0.0.11
#ip netns exec h1ns hping3 -2 -c 1 -q -I veth1 -d 1200 -a 13.0.0.11 -s 2048 -p 2048 2.0.0.12
##ip netns exec h1ns hping3 -2 -q -d 1200 20.0.0.12
##ip netns exec h1ns hping3 -2 -q -d 1495 20.0.0.12 -f -m 1400 -c 1
##ip netns exec h1ns hping3 -2 -q -d 1495 -s 2048 -p 2048 20.0.0.12 -f -m 1400 -c 1



setkey-gw1.conf:
----------------
flush;
spdflush;

add 101.0.0.2 101.0.0.1 esp 0x101 -m tunnel -E aes-cbc 0x89abcdef89abcdef89abcdef89abcdef -A hmac-sha1 0xfedcba98fedcba98fedcba98fedcba98fedcba98;
add 101.0.0.1 101.0.0.2 esp 0x201 -m tunnel -E aes-cbc 0x01234567012345670123456701234567 -A hmac-sha1 0x7654321076543210765432107654321076543210;

spdadd 20.0.0.0/24 30.0.0.0/24 any -P in ipsec
        esp/tunnel/101.0.0.2-101.0.0.1/require;
spdadd 30.0.0.0/24 20.0.0.0/24 any -P out ipsec
        esp/tunnel/101.0.0.1-101.0.0.2/require;


setkey-gw2.conf:
----------------
flush;
spdflush;

add 101.0.0.2 101.0.0.1 esp 0x101 -m tunnel -E aes-cbc 0x89abcdef89abcdef89abcdef89abcdef -A hmac-sha1 0xfedcba98fedcba98fedcba98fedcba98fedcba98;
add 101.0.0.1 101.0.0.2 esp 0x201 -m tunnel -E aes-cbc 0x01234567012345670123456701234567 -A hmac-sha1 0x7654321076543210765432107654321076543210;

spdadd 20.0.0.0/24 30.0.0.0/24 any -P out ipsec
        esp/tunnel/101.0.0.2-101.0.0.1/require;
spdadd 30.0.0.0/24 20.0.0.0/24 any -P in ipsec
        esp/tunnel/101.0.0.1-101.0.0.2/require;



============================================ XXX =======================================

Description:
------------
It may benefit you, if you want to see how Linux stack behaves/generate pcaps even if you don’t have back to back setups.
I am demonstrating  IPsec use-case here.
Concepts/tools I am using, Linux network namespaces, VETH pairs, setkey/ip-xfrm, hping3, ping and tcpdump.

Veth-pairs are as good as back to back connected physical network interfaces.
Send packet from one veth interface, you receive at the other end of the veth pair.
For example, I have created a VETH pair, myeth1 and myeth2. Packets transmit via myeth1 received at myeth2.

Setkey is used for IPsec static SA/SP creation(you can use ip-xfrm as well).
Hping3 is used to send UDP traffic.
Ping is for ICMP(for ppl who are comfortable using ping).
Tcpdump is for sniffing/capturing packets on interface.

I have created 4 network namespaces, namely h1ns, gw1ns, gw2ns and h2ns.
H1ns is host1 namespace.
H2ns is host2 namespace.
Gw1ns is Gateway1 namespace.
Gw2ns is Gateway2 namespace.

Consider these as individual machines(for better understanding).
To execute a command on one namespace(say gw1ns), you have to prepend your cmd with “ip netns exec <ns-name>”
# ip netns exec gw1ns tcpdump -i veth1 -vvv
# ip netns exec gw2ns ifconfig
# ip netns exec h2ns route
I know command is lengthy but worth it.

IPSEC:
------
Host1 <-> Gw1 <=> Gw2 <-> Host2
Just as simple as this. You send plain packet from host1 to host2.
Packet has to be protected until it reaches Gw of host2.
To protect the packet IPSEC security policy and security associations(inbound & outboud) are configured on Gw1 and Gw2.
So the packet gets encrypted at Gw1 and decrypted at Gw2. Then gets forward to the destination.

Attached the setkey configs for gw1 and gw2.
Attached script which does set all the 4 netns ready for use.
The attached shell script has namespace connection topology and IP addresses diagram.

For example, To generate inner-ip fragmentation in ESP packets...
You can either reduce individual interface MTU of the desired interface in the desired namespace or using hping3.
I prefer to using the latter(ie hping3), to send UDP traffic from 30.0.0.11(h1ns) to 20.0.0.12(h2ns).
Why these IPs, refer the system connection/topology diagram in the ipsec-netns-v1.sh

On h1ns# ip netns exec h1ns hping3 -2 -q -d 1495 20.0.0.12 -f -m 1400 -c 1
If you sniff on veth1 of gw1ns, 2 UDP fragments(1400B+95B) should be seen.
If you sniff on veth2 of gw1ns, 2 ESP packets should be seen with no OuterIP fragmented but Inner-IP fragmented.
If you sniff on veth2 of gw2ns, 2 UDP fragments(1400+95) should be seen.
These 2fragments get reassembled at host2
