# Lab 2 – OPNsense Firewall Rules and Traffic Testing

**Author:** Mariam Blessing Ajao

---

## Overview

This lab focused on using OPNsense to create and test firewall rules. I tested how the firewall can block specific types of traffic while allowing other traffic to continue working.

The main tests were blocking ICMP traffic to `1.1.1.1` and blocking outbound HTTP traffic on TCP port `80`, while confirming that HTTPS traffic on TCP port `443` was still allowed.

---

## Lab Environment

| Device           | Description     |
| ---------------- | --------------- |
| Firewall         | OPNsense        |
| Client           | Ubuntu          |
| LAN Network      | `10.10.10.0/24` |
| OPNsense LAN IP  | `10.10.10.1`    |
| Ubuntu Client IP | `10.10.10.167`  |
| Test Destination | `1.1.1.1`       |
| HTTP             | TCP port `80`   |
| HTTPS            | TCP port `443`  |

### Network Diagram

```text
                     Internet
                         |
                  OPNsense Firewall
                  WAN: 10.0.3.x
                  LAN: 10.10.10.1
                         |
                   10.10.10.0/24
                         |
                    Ubuntu Client
                    10.10.10.x
```

---

## Lab Objectives

1. Configure OPNsense firewall rules.
2. Understand firewall rule order.
3. Block specific ICMP traffic.
4. Block outbound HTTP traffic.
5. Confirm HTTPS is still allowed.
6. Examine firewall logs.
7. Observe network traffic using Wireshark.
8. Understand outbound NAT.
9. Restore the firewall configuration after testing.

---

# 1. Baseline Connectivity

Before creating the blocking rules, I tested the network to make sure the connection was working normally.

### Gateway Test

```bash
ping -c 4 10.10.10.1
```

The ping to the OPNsense LAN gateway was successful.

### Internet Ping Test

```bash
ping -c 4 1.1.1.1
```

The ping was successful before the ICMP blocking rule was enabled.

### HTTP Test

```bash
curl --max-time 10 -I http://example.com
```

HTTP traffic was working before the blocking rule was created.

### HTTPS Test

```bash
curl --max-time 10 -I https://example.com
```

HTTPS traffic was also working before the blocking rules were enabled.

---

# 2. Firewall Rule – Block ICMP

I created a firewall rule in:

**Firewall → Rules → LAN**

The rule was configured as follows:

* **Name:** `LAB2 BLOCK ICMP TO 1.1.1.1`
* **Action:** Block
* **Interface:** LAN
* **Direction:** In
* **Protocol:** ICMP
* **Source:** LAN net
* **Destination:** `1.1.1.1`
* **Logging:** Enabled

I placed this rule above the general LAN allow rule.

---

# 3. ICMP Block Test

I tested the rule from the Ubuntu client:

```bash
ping -c 4 1.1.1.1
```

After the rule was enabled, the normal ping replies were blocked.

This confirmed that the OPNsense firewall rule was working.

---

# 4. Firewall Rule – Block Outbound HTTP

I created a second rule to block HTTP traffic.

The rule was configured as:

* **Name:** `LAB2 BLOCK OUTBOUND HTTP`
* **Action:** Block
* **Interface:** LAN
* **Direction:** In
* **Protocol:** TCP
* **Source:** LAN net
* **Destination:** Any
* **Destination Port:** `80`
* **Logging:** Enabled

The rule was placed above the broad LAN allow rule.

---

# 5. HTTP Test

I tested HTTP using:

```bash
curl --max-time 10 -I http://example.com
```

The HTTP request was blocked and timed out.

In Wireshark, I could see TCP retransmissions. This showed that the client was trying to send the HTTP traffic, but the connection was not being completed because the firewall was blocking TCP port `80`.

---

# 6. HTTPS Test

I then tested HTTPS:

```bash
curl --max-time 10 -I https://example.com
```

HTTPS worked successfully and returned an `HTTP/2 200` response.

This showed that blocking TCP port `80` did not block TCP port `443`.

---

# 7. Why Blocking ICMP Did Not Block HTTPS

The ICMP rule only blocks ICMP traffic.

Ping uses **ICMP**, while HTTPS uses **TCP port 443**.

Therefore:

* ICMP → blocked
* TCP 443 → allowed

This showed that firewall rules can be used to control different types of traffic separately.

---

# 8. Why Rule Order Matters

OPNsense checks firewall rules from top to bottom.

The specific blocking rules must be placed above the broad allow rule.

For example:

```text
LAB2 BLOCK ICMP TO 1.1.1.1
LAB2 BLOCK OUTBOUND HTTP
Default allow LAN to any
```

If the broad allow rule comes first, the traffic could be allowed before OPNsense reaches the specific block rule.

---

# 9. Firewall Logs

I enabled logging on the blocking rules.

The firewall logs helped me confirm that traffic was matching the rules and being blocked.

This was useful because it gave me evidence that the firewall was actually processing the traffic according to the rules I created.

---

# 10. Packet Capture

I used Wireshark on the Ubuntu client to observe the network traffic.

I observed different types of packets, including:

* ARP
* ICMP
* DNS
* TCP
* HTTP
* HTTPS

During the HTTP blocking test, I observed TCP retransmissions. This helped me understand what happens to traffic when a firewall rule blocks the connection.

---

# 11. Outbound NAT

The Ubuntu client uses a private IP address:

```text
10.10.10.167
```

This private address cannot directly communicate with the public Internet.

OPNsense performs outbound NAT by translating the private LAN address when traffic goes out through the WAN interface.

In my own simple words, **OPNsense acts like a middleman between my private LAN computer and the Internet.**

---

# 12. Problems Encountered

During the lab, I experienced a few problems while configuring and testing the firewall rules.

### 1. Understanding the Firewall Rule Order

At first, I had to understand why the specific blocking rules needed to be placed above the general allow rule.

I learned that OPNsense checks the rules from top to bottom. If the broad allow rule is above the blocking rule, the traffic may be allowed before reaching the block rule.

### 2. Understanding Why Different Traffic Behaved Differently

I initially had to understand why blocking ICMP did not stop HTTPS.

After testing, I understood that ping uses ICMP, while HTTPS uses TCP port `443`. Therefore, a rule blocking ICMP does not automatically block HTTPS.

### 3. Confirming the HTTP Block

When I blocked HTTP, the `curl` command did not simply show a clear "blocked" message. Instead, the connection timed out.

I used Wireshark to check what was happening and observed TCP retransmissions. This helped me confirm that the traffic was being blocked.

### 4. Making Sure the Rules Were Applied

After creating or changing firewall rules, I needed to make sure the changes were applied before testing again.

I learned that creating a rule is not enough; I must apply the changes and then perform the test again.

### 5. Restoring the Firewall Rules

After completing the tests, I needed to restore the normal network configuration.

I disabled the temporary blocking rules and applied the changes. I then tested the connection again to make sure ping, HTTP, and HTTPS were working normally.

These problems helped me understand how firewall rules work in a practical environment instead of only learning the theory.

---

# 13. Restoration

After completing the tests, I restored the firewall configuration.

I went to:

**Firewall → Rules → LAN**

I disabled the following temporary rules:

* `LAB2 BLOCK ICMP TO 1.1.1.1`
* `LAB2 BLOCK OUTBOUND HTTP`

I then applied the changes.

I disabled the rules instead of deleting them so that I could keep them for future testing and evidence.

---

# 14. Final Restoration Tests

After disabling the temporary rules, I tested the network again.

### Ping

```bash
ping -c 4 1.1.1.1
```

Ping worked again.

### HTTP

```bash
curl --max-time 10 -I http://example.com
```

HTTP worked again.

### HTTPS

```bash
curl --max-time 10 -I https://example.com
```

HTTPS continued to work.

This confirmed that the temporary firewall rules had been successfully disabled.

---

# 15. Test Results

| Test           | Baseline | Block Rules Enabled | Restored |
| -------------- | -------- | ------------------- | -------- |
| Ping `1.1.1.1` | Allowed  | Blocked             | Allowed  |
| HTTP TCP 80    | Allowed  | Blocked             | Allowed  |
| HTTPS TCP 443  | Allowed  | Allowed             | Allowed  |

---

# 16. Key Lessons Learned

From this lab, I learned that:

* Firewall rules control network traffic.
* OPNsense checks rules from top to bottom.
* Specific blocking rules should be placed above broad allow rules.
* ICMP and TCP are different protocols.
* HTTP uses TCP port `80`.
* HTTPS uses TCP port `443`.
* Blocking one port does not automatically block another port.
* Firewall logs can provide evidence of blocked traffic.
* Wireshark can help me see what is happening to network traffic.
* Outbound NAT allows private IP addresses to communicate with the Internet.
* Firewall rules should be tested after making changes.
* Temporary rules can be disabled instead of deleted so they can be reused for future testing.

---

# 17. Analysis Questions

### 1. Why must the specific block rules be placed above the broad allow rule?

The firewall checks rules from top to bottom. If the broad allow rule comes first, the traffic may be allowed before the firewall reaches the specific block rule. That is why the specific block rules should come first.

### 2. Which five packet attributes are most useful when explaining a firewall decision?

The five useful attributes are:

1. Source IP
2. Destination IP
3. Protocol
4. Port number
5. Direction/interface

These help explain where the traffic is coming from, where it is going, what type of traffic it is, and how the firewall handles it.

### 3. Why did blocking ICMP not block HTTPS?

The ICMP rule only blocks ICMP traffic. Ping uses ICMP, while HTTPS uses TCP port `443`.

Therefore, the ping was blocked but HTTPS was still allowed.

### 4. What difference did you observe between the blocked TCP port 80 traffic and permitted TCP port 443 traffic?

The TCP port `80` traffic was blocked and the HTTP connection timed out. Wireshark also showed TCP retransmissions.

TCP port `443` was allowed, and the HTTPS request worked successfully and returned an `HTTP/2 200` response.

### 5. What role does outbound NAT play when the Ubuntu client accesses the Internet?

The Ubuntu client has a private IP address, `10.10.10.167`. Outbound NAT changes the private address when the traffic leaves through the OPNsense WAN interface.

In simple words, OPNsense acts as the middleman between my private LAN computer and the Internet.

### 6. Why should the temporary firewall rules be restored after the lab?

The rules were created for testing. Restoring the firewall prevents the temporary blocks from causing problems later.

It also makes the lab environment ready for the next test and keeps the network working normally.

---

# 18. Evidence

The following screenshots should be included in the `SCREENSHOTS` folder:

```text
01-baseline-connectivity.png
02-icmp-block-rule.png
03-icmp-block-test.png
04-http-block-rule.png
05-http-block-test.png
06-https-allowed-test.png
07-firewall-logs.png
08-wireshark-packet-capture.png
09-disabled-rules.png
10-final-restored-tests.png
```

---

# 19. Conclusion

This lab helped me understand how OPNsense firewall rules control network traffic.

I successfully created rules to block ICMP traffic to `1.1.1.1` and outbound HTTP traffic on TCP port `80`. I also confirmed that HTTPS traffic on TCP port `443` continued to work.

Using firewall logs and Wireshark helped me see evidence of the traffic being blocked. I also learned the importance of firewall rule order and outbound NAT.

Finally, I restored the temporary rules and confirmed that the network was working normally again.
