# Scapy

## 示例代码阅读理解

### ex-1.py

```python
#!/usr/bin/env python

import sys

from scapy.all import *

def PacketHandler(pkt):
    print(pkt.summary())
    return

sniff(iface = sys.argv[1], count = int(sys.argv[2]), prn = PacketHandler)
```

### ex-2.py

```python
#!/usr/bin/env python

import sys

from scapy.all import *

def PacketHandler(pkt):
    if pkt.haslayer(Dot11):
        print(pkt.summary())
    else:
        print("Not an 802.11 pkt")

sniff(iface = sys.argv[1], count = int(sys.argv[2]), prn = PacketHandler)
```

### ex-3.py

```python
#!/usr/bin/env python

import sys
from scapy.all import *

devices = set()

def PacketHandler(pkt):
    if pkt.haslayer(Dot11):
        dot11_layer = pkt.getlayer(Dot11)
        if dot11_layer.addr2 and ( dot11_layer.addr2 not in devices):
            devices.add(dot11_layer.addr2)
            print(len(devices), dot11_layer.addr2, dot11_layer.payload.name)

sniff(iface = sys.argv[1], count  = int(sys.argv[2]), prn = PacketHandler)
```

### ex-4.py

```python
#!/usr/bin/env python

import sys
from scapy.all import *

ssids = set()

def PacketHandler(pkt):
    if pkt.haslayer(Dot11Beacon):
        temp = pkt
        while temp:
            temp = temp.getlayer(Dot11Elt)
            if temp and temp.ID == 0 and (temp.info not in ssids):
                ssids.add(temp.info)
                print(len(ssids), pkt.addr3, temp.info)
                break

            temp = temp.payload

sniff(iface = sys.argv[1], count  = int(sys.argv[2]), prn = PacketHandler)
```

### ex-5.py

```python
#!/usr/bin/env python

import sys
from scapy.all import *

ssids = set()
clientprobes = set()

def PacketHandler(pkt):

    if pkt.haslayer(Dot11ProbeReq):
        if len(pkt.info) > 0:
            testcase = str(pkt.addr2) + '---' + str(pkt.info)
            if testcase not in clientprobes:
                clientprobes.add(testcase)
                print("New Probe Found: " + str(pkt.addr2) + ' ' + str(pkt.info))


                print("\n--------Client Probes Table------\n")
                counter = 1

                for probe in clientprobes:
                    [client, ssid] = probe.split('---')
                    print(counter, client, ssid)
                    counter = counter + 1

                print("\n---------------------------------\n")


sniff(iface = sys.argv[1], count  = int(sys.argv[2]), prn = PacketHandler)
```

### ex-6.py

```python
#!/usr/bin/env python

from scapy.all import *

hidden_ssid_aps = set()

def PacketHandler(pkt):
    if pkt.haslayer(Dot11Beacon):
        if not pkt.info :
            if pkt.addr3 not in hidden_ssid_aps:
                hidden_ssid_aps.add(pkt.addr3)
                print("HIDDEN SSID Network Found! BSSID: ", str(pkt.addr3))

    elif pkt.haslayer(Dot11ProbeResp) and ( pkt.addr3 in hidden_ssid_aps ):
        print("HIDDEN SSID Uncovered!", str(pkt.info), str(pkt.addr3))


sniff(iface = sys.argv[1], count = int(sys.argv[2]), prn = PacketHandler)
```

### ex-7.py

```python
#!/usr/bin/python

import sys
from scapy.all import *

mymac = "aa:aa:aa:aa:aa:aa"
brdmac = "ff:ff:ff:ff:ff:ff"

for ssid in open(sys.argv[1], 'r').readlines():
    pkt = RadioTap() / Dot11(type = 0, subtype = 4, addr1 = brdmac, addr2 = mymac, addr3 = brdmac) / Dot11ProbeReq() / Dot11Elt(ID = 0, info = ssid.strip()) / Dot11Elt(ID = 1, info = "\x02\x04\x0b\x16") / Dot11Elt(ID = 3, info = "\x08")
    print "Trying SSID:", ssid
    sendp(pkt, iface="wlan0mon", count=3, inter = 0.3)
```

### ex-8.py

```python
#!/usr/bin/env python

from __future__ import unicode_literals

import os
import sys
import logging
logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
from scapy.all import Dot11Elt, rdpcap

pcap = sys.argv[1]

if not os.path.isfile(pcap):
    print('input file does not exist')
    exit(1)

beacon_null = set()
ssids_hidden = set()

pkts = rdpcap(pcap)

for pkt in pkts:
    if not pkt.haslayer(Dot11Elt):
        continue
    if pkt.subtype == 8:  # Beacon Frame
        if pkt.getlayer(Dot11Elt).info.decode('utf-8').strip('\x00') == '':
            beacon_null.add(pkt.addr3)
    elif pkt.subtype == 5:  # Probe Response
            if pkt.addr3 in beacon_null:
                ssids_hidden.add(pkt.addr3 + '----' + pkt.getlayer(Dot11Elt).info.decode('utf-8'))

for essid in ssids_hidden:
    print(essid)
```

### ex-9.py

```python
#!/usr/bin/env python

import os
import sys
from scapy.all import Dot11Elt, rdpcap

pcap = sys.argv[1]

if not os.path.isfile(pcap):
    print('input file does not exist')
    exit(1)

pkts = rdpcap(pcap)

ssids = set()
receive_response = {}
probe_request = {}

for pkt in pkts:
    if not pkt.haslayer(Dot11Elt) or pkt.getlayer(Dot11Elt).info.decode('utf-8').strip('\x00') == '':
        continue
    if pkt.subtype == 5: # Probe Response
        if pkt.addr1 not in receive_response:
            receive_response[pkt.addr1] = set()
        receive_response[pkt.addr1].add(pkt.getlayer(Dot11Elt).info.decode('utf-8'))
    if pkt.subtype == 4: # Probe Request
        if pkt.addr2 not in probe_request:
            probe_request[pkt.addr2] = set()
        probe_request[pkt.addr2].add(pkt.getlayer(Dot11Elt).info.decode('utf-8'))

for mac_addr in receive_response.keys():
    if mac_addr in probe_request:
        probe_request.pop(mac_addr)

for mac_addr in probe_request:
    for essid in probe_request[mac_addr]:
        print(essid)
```


## 其他资源

* [Fake wireless Access Point (AP) implementation using Python and Scapy, intended for convenient testing of 802.11 protocols and implementations.](https://github.com/rpp0/scapy-fakeap)


