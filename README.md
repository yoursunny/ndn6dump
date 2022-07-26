# ndn6dump NDN Traffic Dumper

**ndn6dump** is a Go program that captures Named Data Networking network traffic.
It can perform online processing including IP anonymization for privacy protection and NDN packet name extraction.
This program is designed for [yoursunny ndn6 network](https://yoursunny.com/p/ndn6/) but can be used elsewhere.

## Installation

This program is written in Go.
It requires both Go compiler and C compiler.
You can compile and install this program with:

```bash
go install github.com/yoursunny/ndn6dump/cmd/ndn6dump@latest
```

This program is also available as a Docker container:

```bash
docker build -t ndn6dump 'github.com/yoursunny/ndn6dump#main'
```

## Capture Modes

ndn6dump can either live-capture from a network interface via AF\_PACKET socket, or read from a tcpdump trace file.
In both cases, it only recognizes Ethernet link mode.

To live-capture, set the network interface name in `--ifname` flag.
If the NDN forwarder is running in a Docker container, you must run ndn6dump in the same network namespace as the forwarder, and specify the network interface name inside that network namespace.
To capture WebSocket traffic, if the NDN forwarder and the HTTP server that performs TLS termination are communicating over `lo` interface, you must run an additional ndn6dump instance to capture from this interface.
To stop a live capture session, send SIGINT to the ndn6dump process.

To read from a tcpdump trace file, set the filename in `--input` flag and set the local MAC address in `--local` flag.
This mode can recognize `.pcap` `.pcap.gz` `.pcap.zst` `.pcapng` `.pcapng.gz` `.pcapng.zst` file formats.
The local MAC address is necessary for determining traffic direction.

## Output Files

ndn6dump emits two output files.

The **packets** file is an [pcapng](https://datatracker.ietf.org/doc/draft-tuexen-opsawg-pcapng/) file.
It contains Ethernet packets that carry NDN traffic.
IP anonymization has been performed on these packets.
When feasible, NDN packet payload, including Interest ApplicationParameters and Data Content, is zeroized, so that the output can be compressed effectively.

The **records** file is an [Newline delimited JSON (NDJSON)](https://github.com/ndjson/ndjson-spec) file.
Each line in this file is a JSON object that describes a NDN packet, either layer 2 or layer 3.
See [record.go](record.go) for the definition of property keys.
All information in the records file should be available by re-parsing the packets file.

Set output filenames in `--pcapng` and `--json` flags.
If the filename ends with `.gz` or `.zst`, the output file is compressed.

## IP Anonymization

To ensure privacy compliance, ndn6dump performs IP anonymization before output files are written.
IPv4 address keeps its leading 24 bits; IPv6 address keeps its leading 48 bits.
Lower bits are XOR'ed with a random value, which is consistent in each run, so that the same original address yields the same anonymized address.

For WebSocket traffic, HTTP request header `X-Forwarded-For` may contain full client address.
This address is anonymized by changing the lower bits to zeros.

Set IP subnets that should not be anonymized in `--keep-ip` flag (repeatable).
This should be set to subnets used by the network routers, so that it is easier to identify router-to-router traffic.
A side effect is that it would expose non-router IP addresses within the same subnets.
