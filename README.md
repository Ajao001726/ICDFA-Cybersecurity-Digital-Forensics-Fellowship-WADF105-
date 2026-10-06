# Lab 2 – OPNsense Firewall Rules and Traffic Testing

**Author:** Mariam Blessing Ajao

---

## Overview

This lab focused on configuring and testing firewall rules using **OPNsense**.

The main purpose was to understand how a firewall controls network traffic based on the source, destination, protocol, and port.

During the lab, I created firewall rules to:

* Block ICMP traffic to `1.1.1.1`
* Block outbound HTTP traffic on TCP port `80`
* Allow HTTPS traffic on TCP port `443`
* Check firewall logs
* Observe network traffic using packet capture
* Restore the firewall configuration after testing

---

## Lab Environment

| Component        | Details         |
| ---------------- | --------------- |
| Firewall         | OPNsense        |
| Client           | Ubuntu          |
| LAN Network      | `10.10.10.0/24` |
| OPNsense LAN     | `10.10.10.1`    |
| Test Destination | `1.1.1.1`       |
| HTTP             | TCP port `80`   |
| HTTPS            | TCP port `443`  |

### Network Diagram

```text
                    Internet
                       |
                       |
                OPNsense Firewall
                WAN: 10.0.3.x
                LAN: 10.10.10.1
                       |
                       |
                 10.10.10.0/24
                       |
                       |
                  Ubuntu Client
                  10.10.10.x
```

---

## Lab Objectives

The objectives of this lab were to:

1. Configure OPNsense firewall rules.
2. Understand firewall rule order.
3. Block specific ICMP traffic.
4. Block outbound HTTP traffic.
5. Confirm that HTTPS was still allowed.
6. Examine firewall logs.
7. Observe network traffic using Wireshark.
8. Understand the role of outbound NAT.
9. Restore the firewall configuration after testing.

---

# 1. Baseline Connectivity

Before creating the blocking rules, I tested the network to make sure the client had normal connectivity.

### Ping the OPNsense Gateway

```bash
ping -c 4 10.10.10.1
```

The gateway responded successfully.

### Ping the Internet

```bash
ping -c 4 1.1.1.1
```

The ping worked before the ICMP blocking rule was enabled.

### Test HTTP

```bash
curl --max-time 10 -I http://example.com
```

HTTP worked during the baseline test.

### Test HTTPS

```bash
curl --max-time 10 -I https://example.com
```

HTTPS also worked.

These tests confirmed that the network was working before the firewall restrictions were introduced.

**Evidence:** `SCREENSHOTS/01-baseline-connectivity.png`

---

# 2. Firewall Rule – Block ICMP

I created the following rule in:

**Firewall → Rules → LAN**

### Rule Name

```text
LAB2 BLOCK ICMP TO 1.1.1.1
```

### Configuration

| Setting     | Value     |
| ----------- | --------- |
| Action      | Block     |
| Interface   | LAN       |
| Direction   | In        |
| Protocol    | ICMP      |
| Source      | LAN net   |
| Destination | `1.1.1.1` |
| Logging     | Enabled   |

The rule was placed above the general LAN allow rule.

### Why?

OPNsense checks firewall rules from top to bottom. Therefore, the specific block rule needs to be above the broad allow rule.

**Evidence:** `SCREENSHOTS/02-icmp-block-rule.png`

---

# 3. ICMP Block Test

I tested the rule from the Ubuntu client:

```bash
ping -c 4 1.1.1.1
```

The traffic was blocked and the client did not receive normal replies.

This confirmed that the ICMP firewall rule was working.

**Evidence:** `SCREENSHOTS/03-icmp-block-test.png`

---

# 4. Firewall Rule – Block Outbound HTTP

The second rule was created to block normal HTTP traffic.

### Rule Name

```text
LAB2 BLOCK OUTBOUND HTTP
```

### Configuration

| Setting          | Value   |
| ---------------- | ------- |
| Action           | Block   |
| Interface        | LAN     |
| Direction        | In      |
| Protocol         | TCP     |
| Source           | LAN net |
| Destination      | Any     |
| Destination Port | `80`    |
| Logging          | Enabled |

The rule was also placed above the broad LAN allow rule.

**Evidence:** `SCREENSHOTS/04-http-block-rule.png`

---

# 5. HTTP Test

I tested HTTP using:

```bash
curl --max-time 10 -I http://example.com
```

The connection was blocked or failed to complete.

This confirmed that TCP port `80` traffic was being blocked.

In Wireshark, I also observed **TCP retransmissions**, showing that the connection was trying again because it was not getting a successful response.

**Evidence:** `SCREENSHOTS/05-http-block-test.png`

---

# 6. HTTPS Test

I then tested HTTPS:

```bash
curl --max-time 10 -I https://example.com
```

The HTTPS connection worked successfully and returned **HTTP/2 200**.

This was expected because HTTPS uses TCP port `443`, while the firewall rule was only blocking TCP port `80`.

```text
HTTP  → TCP 80  → BLOCKED
HTTPS → TCP 443 → ALLOWED
```

**Evidence:** `SCREENSHOTS/06-https-allowed-test.png`

---

# 7. Why Blocking ICMP Did Not Block HTTPS

The ICMP rule only affected ICMP traffic.

Ping uses ICMP:

```text
ping → ICMP
```

HTTPS uses TCP:

```text
HTTPS → TCP 443
```

Therefore, blocking ICMP did not stop HTTPS.

### In my own words

The firewall was only blocking ping traffic. HTTPS uses TCP instead of ICMP, so the HTTPS connection could still pass through the firewall.

---

# 8. Why Rule Order Matters

OPNsense processes rules from the top down.

A broad rule such as:

```text
Allow LAN net → Any
```

can allow a large amount of traffic.

Therefore, the specific block rules had to be placed above it.

### In simple words

The firewall reads the rules from the top. If the general allow rule comes first, it can allow the traffic before the firewall reaches my specific block rule.

---

# 9. Firewall Logs

Logging was enabled on the blocking rules.

The logs helped confirm that the traffic matched the firewall rules.

Five useful packet attributes for understanding a firewall decision are:

1. **Source IP address** – where the traffic is coming from.
2. **Destination IP address** – where the traffic is going.
3. **Protocol** – for example, TCP, UDP, or ICMP.
4. **Port number** – for example, port 80 for HTTP and port 443 for HTTPS.
5. **Direction/interface** – where the traffic is entering or passing through the firewall.

These details help me understand **why the firewall allowed or blocked the traffic**.

**Evidence:** `SCREENSHOTS/07-firewall-logs.png`

---

# 10. Packet Capture

Wireshark was used to observe network traffic.

The packet capture helped identify different types of traffic, including:

* ARP
* ICMP
* DNS
* TCP
* HTTP
* HTTPS

Packet capture provided additional evidence of what was happening on the network.

**Evidence:** `SCREENSHOTS/08-wireshark-packet-capture.png`

---

# 11. Outbound NAT

The Ubuntu client uses a private IP address:

```text
10.10.10.167
```

Outbound NAT allows the client to access the Internet by translating its private IP address through the OPNsense WAN address.

```text
Ubuntu Client
10.10.10.167
      |
      v
OPNsense LAN
10.10.10.1
      |
      | NAT
      v
OPNsense WAN
10.0.3.x
      |
      v
Internet
```

### In simple words

Outbound NAT allows the private IP address of the client to communicate with the Internet by translating the address as the traffic leaves the firewall.

OPNsense acts like a middleman between my private LAN computer and the Internet.

---

# 12. Restoration

After completing the tests, I restored the firewall.

I went to:

**Firewall → Rules → LAN**

I disabled:

```text
LAB2 BLOCK ICMP TO 1.1.1.1
```

and:

```text
LAB2 BLOCK OUTBOUND HTTP
```

I did not delete the rules because they were required as evidence.

I then applied the changes.

**Evidence:** `SCREENSHOTS/09-disabled-rules.png`

---

# 13. Final Restoration Tests

After disabling the blocking rules, I tested the connections again.

### ICMP

```bash
ping -c 4 1.1.1.1
```

The ping worked again.

### HTTP

```bash
curl --max-time 10 -I http://example.com
```

HTTP worked again.

### HTTPS

```bash
curl --max-time 10 -I https://example.com
```

HTTPS also worked.

This confirmed that the laboratory had been successfully restored.

**Evidence:** `SCREENSHOTS/10-final-restored-tests.png`

---

# 14. Test Results

| Test           | Baseline  | Block Rules Enabled | Restored  |
| -------------- | --------- | ------------------- | --------- |
| Ping `1.1.1.1` | ✅ Allowed | ❌ Blocked           | ✅ Allowed |
| HTTP TCP 80    | ✅ Allowed | ❌ Blocked           | ✅ Allowed |
| HTTPS TCP 443  | ✅ Allowed | ✅ Allowed           | ✅ Allowed |

---

# 15. Key Lessons Learned

From this lab, I learned that:

* Firewall rules control network traffic.
* OPNsense checks rules from top to bottom.
* Specific rules should be placed above broad rules.
* ICMP and TCP are different protocols.
* HTTP uses TCP port 80.
* HTTPS uses TCP port 443.
* Blocking one port does not automatically block another port.
* Firewall logs help confirm blocked traffic.
* Wireshark helps observe network packets.
* NAT allows private IP addresses to communicate with the Internet.
* Firewall rules should be tested after configuration changes.
* Temporary rules can be disabled instead of deleted when evidence needs to be preserved.

---

# 16. Analysis Questions

### 1. Why must the specific block rules be placed above the broad allow rule?

The firewall checks the rules **from top to bottom**. If the block rule is below the general allow rule, the traffic may be allowed before the firewall reaches the block rule.

So, I placed the specific block rules **above the “Default allow LAN to any” rule** so that the traffic I wanted to block would be stopped first.

---

### 2. Which five packet attributes are most useful when explaining a firewall decision?

The five important things are:

1. **Source IP** – where the traffic is coming from.
2. **Destination IP** – where the traffic is going.
3. **Protocol** – for example, TCP, UDP, or ICMP.
4. **Port number** – for example, port 80 for HTTP and port 443 for HTTPS.
5. **Direction/interface** – where the traffic is entering or passing through the firewall.

These help me understand **why the firewall allowed or blocked the traffic**.

---

### 3. Why did blocking ICMP not block HTTPS?

Blocking ICMP did not block HTTPS because they are **different types of traffic**.

My rule was only blocking **ICMP traffic to 1.1.1.1**. HTTPS uses **TCP on port 443**, so it was not affected by the ICMP block.

I tested this and the ping failed, but HTTPS still worked.

---

### 4. What difference did you observe between the blocked TCP port 80 traffic and permitted TCP port 443 traffic?

When I tested **HTTP on port 80**, the connection was blocked and the request timed out.

In Wireshark, I also saw **TCP retransmissions**, showing that the connection was trying again because it was not getting a successful response.

When I tested **HTTPS on port 443**, it worked successfully and returned **HTTP/2 200**.

So, **port 80 was blocked while port 443 was allowed**.

---

### 5. What role does outbound NAT play when `icdfa-nslab-client-v1` uses a private IPv4 address?

My client has a private IP address, `10.10.10.167`.

Outbound NAT allows the client to **access the Internet by translating its private IP address through the OPNsense WAN address**.

In simple terms, OPNsense acts like a middleman between my private LAN computer and the Internet.

---

### 6. Why is restoring the original state an important part of a controlled security laboratory?

Restoring the original settings is important because the changes I made were mainly for testing.

After the lab, restoring the original state makes sure that my temporary firewall rules **do not cause problems later**.

It also makes the lab easier to repeat and ensures that the network is left in a **safe and expected condition**.

---

# 17. Evidence

The following screenshots are included in the `SCREENSHOTS` folder:

| No. | Evidence                 | File                              |
| --: | ------------------------ | --------------------------------- |
|   1 | Baseline connectivity    | `01-baseline-connectivity.png`    |
|   2 | ICMP block rule          | `02-icmp-block-rule.png`          |
|   3 | ICMP blocked test        | `03-icmp-block-test.png`          |
|   4 | HTTP block rule          | `04-http-block-rule.png`          |
|   5 | HTTP blocked test        | `05-http-block-test.png`          |
|   6 | HTTPS allowed test       | `06-https-allowed-test.png`       |
|   7 | Firewall logs            | `07-firewall-logs.png`            |
|   8 | Wireshark packet capture | `08-wireshark-packet-capture.png` |
|   9 | Disabled rules           | `09-disabled-rules.png`           |
|  10 | Final restored tests     | `10-final-restored-tests.png`     |

---

# 18. Conclusion

This lab gave me practical experience configuring and testing firewall rules using OPNsense.

I created a rule to block ICMP traffic to `1.1.1.1` and another rule to block outbound HTTP traffic on TCP port `80`.

The tests showed that the firewall could block specific traffic while allowing other traffic. For example, blocking HTTP did not block HTTPS because HTTPS uses TCP port `443`.

I also used firewall logs and packet capture to understand and verify network traffic.

Finally, I disabled the temporary blocking rules and confirmed that ping, HTTP, and HTTPS worked again.

This lab helped me understand how firewall rules, protocols, ports, rule order, logging, packet capture, and NAT work together to control network communication.
