---
title: "Intrusion - Scada"
author: "xThaz"
context: "HTB Business 2023"
tags : ["Write-Up", "HTB"]
---
# Intrusion - SCADA

On a un pcap avec toujours la même séquence de réponses Modbus.
![](https://i.imgur.com/WDDUQbu.png)

On récupère les addresses auxquelles on a écrit avec **Write Multiple Registers (16)**? (je sais pas pourquoi, me demandez pas).

```python
from pyModbusTCP.client import ModbusClient

c = ModbusClient(host='94.237.48.19', port=33199, unit_id=52, auto_open=True)

regs = [6,10,12,21,22,26,47,53,63,77,83,86,89,95,96,104,123,128,131,134,139,143,144,145,153,163,168,173,179,193,206,210,214,215,219,221,224,225,226,231,239,253] # Recovered from pcap
flag = ""
for reg in regs:
    d = c.read_holding_registers(reg, 1)[0]
    flag += chr(d)

print(flag)
```
