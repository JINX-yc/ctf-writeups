# Which Cap?

**Category:** Forensics  
**Platform:** Netanix / NxCTF  

---

## Overview

We are given a ZIP containing a `.pcap` file. The flag is hidden using IP steganography — data is encoded in the destination IP addresses of ICMP packets.

---

## Solution

### Step 1 — Open the PCAP in Wireshark

Extract the ZIP and open the capture file in Wireshark.

Initial observations:
- Many DNS requests to `google.com` subdomains that look base64-encoded, but decoding them yields nothing useful
- A plaintext fake flag is visible: `softwarica{n0t_th3_r3al_pcap_fl4g}`

### Step 2 — Filter by protocol

In Wireshark, go to **Statistics → Protocol Hierarchy** and filter for **ICMP** traffic.

The ICMP packets stand out as suspicious — too many of them with seemingly random destination IPs.

### Step 3 — Research the encoding technique

After investigating, the challenge uses **IP steganography** — characters of the flag are encoded in the destination IP addresses of ICMP packets during transmission.

### Step 4 — Extract destination IPs with tshark

```bash
tshark -r challenge.pcap -Y "icmp" -T fields -e ip.dst > ips.txt
```

- `-r challenge.pcap` — read from the capture file
- `-Y "icmp"` — filter to show only ICMP packets
- `-T fields` — output selected fields only (no verbose output)
- `-e ip.dst` — extract only the destination IP address per packet
- `> ips.txt` — save to file

### Step 5 — Decode the IPs to recover the flag

Write (or prompt an AI to generate) a Python script that reads `ips.txt` and converts each IP's last octet (or relevant byte) into an ASCII character to reconstruct the flag.

```python
# Example decoder — adjust based on actual encoding scheme
with open("ips.txt") as f:
    ips = f.read().splitlines()

flag = ""
for ip in ips:
    parts = ip.split(".")
    flag += chr(int(parts[-1]))

print(flag)
```

Running the script outputs the flag.

---

## Key Insight

IP steganography embeds data in fields of IP packets (such as the last octet of the destination address) that are not semantically significant to routing but are preserved during transmission. `tshark` makes it easy to extract these fields in bulk for analysis.

---

## Flag

> Recovered from decoded IP destination octets in the ICMP traffic.
