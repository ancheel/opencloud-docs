# Gateway LAN communication protocol V2.0

## **Overview**

LUMI Smart Gateway supports LAN communication features. Through opening the LAN communication API, developers are allowed to manage each ZigBee sub-device (sensor, controller, etc.) under the gateway via LAN communication.

Compared to HTTP communication, LAN communication is faster and has lower control latency. However, the development cost of a LAN connection is higher. Furthermore, it requires a third-party gateway that supports development and the developer has experience in embedded development.

Currently, the main API features opened are:

- Discover and query devices
- Report device status
- Read and write operations on devices
- Report device heartbeat

## **Revision record**

Introduces the major changes for each version of the Gateway LAN protocol.

| **Update time** | **Document version** | **Update log**                           |
| --------------- | -------------------- | ---------------------------------------- |
| 2018.05.18      | V2.0.1               | Add: Cube (sensor_cube.aqgl01), Wall Outlet (ctrl_86plug.aq1), Wall Switch(ctrl_ln1.aq1) |
| 2017.10.09      | V2.0.1               | Modifications: support property report on water leak sensor |
| 2017.09.20      | V2.0.0               | Modifications: Changes to the basic JSON format; device model value and property name change for some devices; for the property value type, each analog value is assigned to a corresponding numerical value. |

## **Encryption mechanism**

The key encryption method is applied in LAN communication, and the APP needs to obtain a randomly generated gateway KEY. The gateway KEY is encrypted by AES-CBC 128 and is a string of 16 bytes in length. Can only use Gateway LAN Communication after activating the the LAN Communication protocol and obtaining the key for the gateway. 

> **Note:** That the initial vector of the AES-CBC 128 is defined as: unsigned char const AES_KEY_IV [16] = {0x17, 0x99, 0x6d, 0x09, 0x3d, 0x28, 0xdd, 0xb3, 0xba, 0x69, 0x5a, 0x2e, 0x6f, 0x58, 0x56, 0x2e}.



Specific operations for obtaining the gateway KEY are as follows:

1.Open Aqara APP, select the gateway device that needs LAN communication.

> **Note:** Only **Air Conditioning Controller(advanced)** supports LAN communication.

2.This page does not show "LAN Communication Protocol" by default. You need tap on "device type" 10 times and it will appear. 

3.Enable "LAN Communication Protocol" to get a random KEY. Click "OK".



## **Discovery and Query Devices**

### **Discover the gateway device**

The device discovery operates in unencrypted mode and use **multicast** (**IP: 224.0.0.50  peer_port: 4321 protocal: UDP**) to discover the gateway device on the LAN.

Send the "whois" command in multicast mode:

```
{
   "cmd":"whois"
}
```

All gateways that receive the "whois" command are required to reply with their own IP information in **unicast** mode:

```
{
   "cmd":"iam",
   "ip":"192.168.0.42",   //Gateway IP address
   "protocal":"UDP",
   "port":"9898",
   "model":"gateway.aq1",  //Gateway device model
   ......
}
```

### **Query sub-device list**

The command is sent via **unicast** to the **UDP 9898** port of the gateway, which is used to obtain the sub-devices in the gateway.

Send a "discovery" command to the gateway via unicast:

```
{
   "cmd":"discovery"
}
```

The gateway replies by unicast, returning the device id and model value of the sub-devices:

```
{
   "cmd":"discovery_rsp",
   "sid":"158d323123c9d9",     //sid is gateway did
   "token":"TahkC7dalbIhXG22",    //random string generated by the gateway
   "dev_list":[{"sid":"158d0000f1a750","model":"plug"},  
               {"sid":"158d00010fd645","model":"sensor_switch.aq2"}]  //sid is sub-device did
}
```

> **Note**: "token" is a random string generated by the gateway. It is refreshed every 10s. Before the token is received from the device according to heartbeat of the device, the user can use this token to generate a "key" for writing to the device.

## **Report device status**

When the device status changes, the "report" command is sent via **multicast** to (**IP: 224.0.0.50 Port: 9898**) to report the status attribute such as the opening or closing information detected by the door and window sensors. Using the attribute status that are reported, the user can create related intelligent operations, such as turning on the air conditioner when the windows are closed.

Example:

Door and window sensors reported the open/close status of windows, the format is as follows:

```
{
   "cmd":"report",
   "model":"sensor_magnet.aq2",
   "sid":"158d0000123456",
   "params":[{"window_status":"open"}] 
}
```

## **Report device heartbeat**

**Gateway heartbeat**

The gateway heatbeat is sent via **multicast** to **(IP: 224.0.0.50 Port: 9898**). The gateway sends a heartbeat packet every 10 seconds to inform the PC gateway that it is working normally. If the heartbeat packet is not received after more than 65 seconds, the status of the gateway set to off-line. Heartbeat format of the gateway device is as follows:

```
{
    "cmd":"heartbeat",
    "model":"gateway.v3",
    "sid":"f0b429b3c9d965",
    "token":"1234567890abcdef",   //random string generated by the gateway
    "params":[{"ip":"172.22.4.130"}]  //Gateway IP address
 }
```

> **Note**: "token" is a random string generated by the gateway. It is refreshed every 10s. This token can be used to generate the "key" when writing to the device.

**Heartbeat of Sub-devices**

The heartbeat of sub-devices are sent via **multicast** to (**IP: 224.0.0.50 Port: 9898**). The sub-device tells the PC via the heartbeat that the sub-device is working normally (heartbeat reporting frequency: generally devices in sleep mode report once every hour, and devices that are plugged in to electrical outlets report once every 10 minutes). The heartbeat format of sub-devices is as follows:

```
{
   "cmd":"heartbeat",
   "model":"sensor_magnet.aq2",
   "sid":"158d000065a271",
   "params":[{"window_status":"open"}]
}
```

The heartbeat of the sub-device may contain the attribute status of the sub-device, such as  `"window_status": "open"`. When setting the heartbeat, consideration need to be given for the statuses of usage scenarios.

For example: Open window and close air conditioning scenario, you can use the above heartbeat to trigger actions. But in the close window and turn on air conditioning scenario, you can not use the above heartbeat. This is because it is possible that the air conditioner was turned off when the person leaves, but the heartbeat message turns on the air conditioner with nobody in the room, so it is a waste of electricity.

Therefore, when using heartbeat messages, the user can decide whether to use the heartbeat to trigger certain actions according to the user's usage needs.

## **Read and write operations of the device**

**Reading devices**

Send the "read" command via **unicast** to the gateway's UDP 9898 port. Users can take the initiative to read the attribute status of each device, and the gateway returns all the attribute information associated with the device.

Example:

Read the status of the wall switch:

```
{
   "cmd":"read",
   "sid":"158d0000123456"   //wall switch did
}
```

Gateway sends reply via **unicast**, the format is as follows:

```
{
   "cmd":"read_rsp",
   "model":"ctrl_neutral2",
   "sid":"158d0000123456",
   "params":[{"channel_0":"on"},{"channel_1":"off"}]  
}
```

**Writing devices**

Send the "write" command via **unicast** to the gateway's UDP 9898 port. When users need to control the device, the user can use the "write" command.

Example:

Change the status of the wall switch (With Neutral, Single Rocker) to off:

```
{
    "cmd":"write",
    "model":"ctrl_neutral1",
    "sid":"158d0000123456",
    "key":"3EB43E37C20AFF4C5872CC0D04D81314",
    "params":[{"channel_0":"off"}]
 }
```

Gateway sends reply via **unicast**, the format is as follows:

```
{
   "cmd":"write_rsp",
   "model":"ctrl_neutral1",
   "sid":"158d0000123456",
   "params":[{"channel_0":"on"}]  
}
```

This "write_rsp" only represents that the gateway has received the write command. The attribute status in params is the current status of the current device, not the final device status after the write operation. The final device status is reported by "report" message.

> **Note**: "key" is a 32-byte string. When the encryption mode is enabled on the gateway, the key is decrypted and validated to verify the validity of the write command. The "key" generation rule is as follows: After receiving the 16-byte "token" string in the heartbeat, the user uses the KEY of the gateway (random KEY obtained in the APP) to perform AES-CBC 128 encryption, generating 16 bytes of ciphertext. The ciphertext is then converted to a 32-bytes ASCII coded string.
>
> For example, the random KEY with a length of 16 characters in the Mi Smart Home APP is "0987654321qwerty", the token is "1234567890abcdef", and the encrypted ciphertext is 0x3E,0xB4,0x3E,0x37,0xC2,0x0A,0xFF,0x4C,0x58,0x72,0xCC,0x0D,0x04,0xD8,0x13,0x14. So the "key" is: "3EB43E37C20AFF4C5872CC0D04D81314".

## **Device reporting and control message format**

JSON message basic format:

```
{
   "cmd":"write",     //command type(write/read/write_rsp/read_rsp/report/heartbeat)
   "model":"ctrl_neutral2",   //device type
   "sid":"158d0000123456",    //device did
   "params":[{"channel_0":"on"},{"channel_1":"off"}]  
   //params can contain multiple properties for the same device
}
```

## **Device Properties List**

Describes device types, properties, and usage examples for Aqara products.

### **Air Conditioning Controller**

(device model：acpartner.v3)

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| illumination    | The illumination of the Air Conditioning Controller, has a general value range of 0 ~ 1300; supports report/read. |
| proto_version   | Communication protocol version number used, such as "2.0.1". |
| mid             | Indicates music id, the id of the music ringtone. Supports write. The values are: 0 ~ 8, 10 ~ 13, and 20 ~ 29 (the systems mentioned above comes with ringtones), 10000 (stop ringing), >10001 (user-defined ringtones). |
| join_permission | The value of "yes"/"no" is used to indicate whether sub-devices can be added. |
| remove_device   | The value is a did (did hex) string of the sub-device. It is used to remove a sub-device. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"acpartner.v3",
   "sid":"640980123456",
   "params":["illumination:500,"proto_version":"2.0.0"}]
   //illumination is 500，the communication protocol version is 2.0.0.
}
```

Control:

Play custom ringtones (mid value 10005):

```
{
   "cmd":"write",
   "model":"acpartner.v3",
   "sid":"640980123456",
   "params":[{"mid":10005}]
}
```

Stop playing ringtones: 

```
{
   "cmd":"write",
   "model":"acpartner.v3",
   "sid":"640980123456",
   "params":[{"mid":10000}]
}
```

Allow adding sub-devices:

```
{
   "cmd":"write",
   "model":"acpartner.v3",
   "sid":"640980123456",
   "params":[{"join_permission":"yes"}]
}
```

> **Note**: The operation to add a sub-device needs to be completed within 30s. Press and hold the sub-device reset key for 3 seconds until the blue LED flashes continuously. When the gateway prompts that the device has been added successfully, the add to network operation is successful. The indicator may not be the same on different sub-devices when press and hold the reset key. Please operate the sub-device according to the actual situation.



Delete a sub-device under the Air Conditioning Controller:

```
{
   "cmd":"write",
   "model":"acpartner.v3",
   "sid":"640980123456",
   "params":[{"remove_device":"158d0000f12345"}]
}
```

### **Plug**

(device model：plug)

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| channel_0       | on / off                                 |
| load_power      | Load power in watts (W)                  |
| energy_consumed | The cumulative load power consumption since the product was used, in watt-hours (Wh) |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"plug",
   "sid":"158d0000123123",
   "params":[{"channel_0":"on"}] //Plug status is "ON".
}
```

Control: 

```
{
   "cmd":"write",
   "model":"plug",
   "sid":"158d0000123123",
   "params":[{"channel_0":"off"}]  //Change plug status to "OFF".
}
```

Heartbeat report (~ 10 minutes each time):

```
{
   "cmd":"heartbeat",
   "model":"plug",
   "sid":"158d0000123123",
   "params":[{"load_power":9.57},{”energy_consumed":57}] 
   //Load power is 9.57W, the load consumed 57Wh. 
}
```

### **Wall Outlet**

(device model：ctrl_86plug and ctrl_86plug.aq1)

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| channel_0       | on / off / unknown                       |
| load_power      | Load power in watts (W)                  |
| energy_consumed | The cumulative load power consumption since the product was used, in watt-hours (Wh) |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"ctrl_86plug",
   "sid":"158d0001123123",
   "params":[{"channel_0":"on"}]  //Wall outlet status is "ON".
}
```

Control: 

```
{
   "cmd":"write",
   "model":"ctrl_86plug",
   "sid":"158d0001123123",
   "params":[{“channel_0”:”off”}]  //Change wall outlet status to "OFF".
}
```

Heartbeat report (~ 10 minutes each time):

```
{
   "cmd":"heartbeat",
   "model":"ctrl_86plug",
   "sid":"158d0001123123",
   "params":[{"load_power":9.57},{"energy_consumed":57}]
   //Load power is 9.57W, the load consumed 57Wh. 
}
```

### **Wall Switch (With Neutral, Single Rocker)**

(device model：ctrl_ln1 and ctrl_ln1.aq1)

| **Attributes** | **Description** |
| -------------- | --------------- |
| channel_0      | on/off/unknown  |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"ctrl_ln1",
   "sid":"158d0001112313",
   "params":[{"channel_0":"on"}]  //Wall switch status is "ON".
}
```

Control: 

```
{
   "cmd":"write",
   "model":"ctrl_ln1",
   "sid":"158d0001112313",
   "params":[{"channel_0":"off"}]   //Change the wall switch status to "OFF".
}
```

### **Wall Switch (With Neutral, Double Rocker)**

(device model：ctrl_ln2)

| **Attributes** | **Description**    |
| -------------- | ------------------ |
| channel_0      | on / off / unknown |
| channel_1      | on / off / unknown |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"ctrl_ln2",
   "sid":"158d0001112312",
   "params":[{"channel_1":"on"}]  
}
```

Control: 

```
{
   "cmd":"write",
   "model":"ctrl_ln2",
   "sid":"158d0001112312",
   "params":[{"channel_1":"off"}] 
}
```

### **Wall Switch (No Neutral, Single Rocker)**

(device model：ctrl_neutral1)

| Attributes | Description |
| ---------- | ----------- |
| channel_0  | on/off      |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"ctrl_neutral1",
   "sid":"158d0001112311",
   "params":[{"channel_0":"on"}]  
}
```

Control: 

```
{
   "cmd":"write",
   "model":"ctrl_neutral1",
   "sid":"158d0001112311",
   "params":[{" channel_0":"off"}]  
}
```

### **Wall Switch (No Neutral, Double Rocker)**

(device model：ctrl_neutral2)

| Attributes | Description |
| ---------- | ----------- |
| channel_0  | on/off      |
| channel_1  | on/off      |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"ctrl_neutral2",
   "sid":"158d0001112316",
   "params":[{"channel_0":"on"}]  
}
{
   "cmd":"report",
   "model":"ctrl_neutral2",
   "sid":"158d0001112316",
   "params":[{"channel_1":"on"}]  
}
```

 Control: 

```
{
   "cmd":"write",
   "model":"ctrl_neutral2",
   "sid":"158d0001112316",
   "params":[{"channel_0":"on"}]  
}
{
   "cmd":"write",
   "model":"ctrl_neutral2",
   "sid":"158d0001112316",
   "params":[{"channel_1":"off"}]  
}
```

### **Curtain Controller**

(device model：curtain)

| **Attributes** | **Description**                          |
| -------------- | ---------------------------------------- |
| curtain_status | open / close / stop / auto (open the curtain / close the curtain / stop operations / automated operations). Supports "write" attribute (reports curtain_level after writing to device), does not support "report". |
| curtain_level  | Value: 0-100 indicates the percentage that a curtain is open; -1 or 255 indicates that the position of the curtain is unknown. Support "write" and "report". |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"curtain",
   "sid":"158d0001012345",
   "params":[{"curtain_level":50}]  
}
```

Control: 

```
{
   "cmd":"write",
   "model":"curtain",
   "sid":"158d0001012345",
   "params":[{"curtain_status":"open"}]  
}
{
   "cmd":"write",
   "model":"curtain",
   "sid":"158d0001012345",
   "params":[{"curtain_level":25}]  
}
```

Report curtain open status:

```
{
   "cmd":"report",
   "model":"curtain",
   "sid":"158d0001012345",
   "params":[{"curtain_level":25}]  
}
```

### **Wireless Relay Controller (2 Channels)**

（device model：lumi.ctrl_dualchn）

| Attributes | Description            |
| ---------- | ---------------------- |
| channel_0  | on/off/toggle (on/off) |
| channel_1  | on/off/toggle (on/off) |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"lumi.ctrl_dualchn",
   "sid":"158d0001112316",
   "params":[{"channel_0":"on"}] 
}
```



### **Door and Window Sensor**

(device model：sensor_magnet.aq2)

Doors and windows sensors detect the status of whether windows or doors are open/closed. The report is sent after every request/action.

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| window_status   | open / close / unknown                   |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"sensor_magnet.aq2",
   "sid":"158d0001123456",
   "params":[{"window_status":"open"}]  
}
```

Heartbeat report (~ 60 minutes each time):

```
{
   "cmd":"report",
   "model":"sensor_magnet.aq2",
   "sid":"158d0001123456",
   "params":[{"battery_voltage":3000}]
}
```

### **Motion Sensor**

(device model：sensor_motion.aq2)

The motion sensor detects someone moving and immediately reports this information. At the same time, it also reports illumination values "lux" and "illumination". In the case of someone moving, in order to save electricity, the motion sensor sends a report once every minute. The motion sensor also reports the current lux value for each heartbeat. In other cases, the motion sensor does not report the light value.

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |
| motion_status   | Value: motion indicates that someone was detected; unknown indicates unknown |
| lux             | Illumination value. The value ranges 0 to 1200. When it detects the movement of people, the illumination of the lighting is collected and reported; or reported as part of the sensor's heartbeat. |
| illumination    | Illumination value. The value ranges 0 to 1200 value, Only collects illumination and report when someone moves. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"sensor_motion.aq2",
   "sid":"158d0000f12222",
   "params":[{"motion_status":"motion"}]  
}
```

 Report illumination value at the same time:

```
{
   "cmd":"report",
   "model":"sensor_motion.aq2",
   "sid":"158d0000f12222",
   "params":[{"lux":100},{"illumination":100}]  
}
```

Heartbeat report (~ 60 minutes each time):

```
{
   "cmd":"report",
   "model":"sensor_motion.aq2",
   "sid":"158d0000f12222",
   "params":[{"battery_voltage":3000},{"lux":50}]
}
```

### **Temperature and Humidity Sensor**

(device model：weather)

When the temperature and humidity sensor detects temperature changes over 0.5 degrees or more than 6% humidity change, a report will be sent. The atmospheric pressure value will be sent along with the temperature or humidity report. The Temperature and humidity sensor also reports the current temperature, humidity, and atmospheric pressure values during each heartbeat.

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| temperature     | Temperature, numeric, default invalid value is set to 10000 |
| humidity        | Humidity, numeric, the default invalid value is set to 0 |
| pressure        | Atmospheric pressure value, numerical, unit Pa, value range 30000 ~ 110000 The default invalid value is 0 |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"weather",
   "sid":"158d0000f12346",
   "params":[{"temperature":2333}]  //Temperature is 23.33 degrees.
}
```

Heartbeat report (~ 60 minutes each time):

```
{
   "cmd":"heartbeat",
   "model":"weather",
   "sid":"112316",
   "params":[{"battery_voltage":3000}, {"temperature":2333},{"humidity":6678},{"pressure":99900}]
   //Temperature is 23.33 degrees, humidity is 66.78%, atmospheric pressure is 99.9KPa.
}
```

### **Water Leak Sensor**

(device model：sensor_wleak.aq1)

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| wleak_status    | Value: "normal" means no alarm or alarm has been deactivated; "leak" means water leak alarm. |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"sensor_wleak.aq1",
   "sid":"158d0001234567",
   "params":[{"wleak_status":"leak"}]  
}
```

Heartbeat report (~ 60 minutes each time):

```
{
   "cmd":"heartbeat",
   "model":"sensor_wleak.aq1",
   "sid":"158d0001234567",
   "params":[{"battery_voltage":3000}]
}
```

### **Wireless Mini Switch**

(device model：sensor_switch.aq2)

The Wireless Mini Switch reports once for each click, it reports twice when clicked on twice within 400ms.

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| button_0        | click / double_click                     |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"sensor_switch.aq2",
   "sid":"158d0000123456",
   "params":[{"button_0":"click"}]  
}

```

Heartbeat report (~ 60 minutes each time):

```
{
   "cmd":"heartbeat",
   "model":"sensor_switch.aq2",
   "sid":"158d0000123456",
   "params":[{"battery_voltage":3000}]
}
```



### **Wireless Mini Switch**

(device model：sensor_switch.aq3)

The Wireless Mini Switch reports once for each click, it reports twice when clicked on twice within 400ms.

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| button_0        | click / double_click                     |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"sensor_switch.aq3",
   "sid":"158d0000123456",
   "params":[{"button_0":"click"}]  
}
```

Heartbeat report (~ 60 minutes each time):

```
{
   "cmd":"heartbeat",
   "model":"sensor_switch.aq3",
   "sid":"158d0000123456",
   "params":[{"battery_voltage":3000}]
}
```

### **Wireless Remote Switch (Single Rocker)**

(device model：sensor_86sw1.aq1)

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| button_0        | click / double_click (click / double click) |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"sensor_86sw1.aq1",
   "sid":"158d0000123456",
   "params":[{"button_0":"double_click"}]
}
```

### **Wireless Remote Switch (Double Rocker)**

(device model：sensor_86sw2.aq1)

| **Attributes** | **Description**                          |
| -------------- | ---------------------------------------- |
| button_0       | click (left-click); double_click (left double-click) |
| button_1       | click (right-click); double_click (right double-click) |
| dual_channel   | both_click (left and right keys clicked simultaneously) |

Example:

Property Report:

```
{
   "cmd":"report",
   "model":"sensor_86sw2.aq1",
   "sid":"158d0000123456",
   "params":[{"button_1":"double_click"}]
}
```

### **Cube**

(device model：sensor_cube.aqgl01)

| **Attributes**  | **Description**                          |
| --------------- | ---------------------------------------- |
| cube_status     | flip90/flip180/move/tap_twice/shake_air/swing/alert/free_fall/rotate |
| rotate_degree   | Rotation angle, the unit is degree(°) . Positive numbers indicate clockwise rotations and negative numbers indicate counterclockwise rotations. |
| detect_time     | Rotation time, the unit is ms.           |
| battery_voltage | Button battery voltage, measured in mv units, range 0~3300mv. Under normal circumstances, less than 2800mv is considered to be low battery power. |

Example:

Property Report:

Rotation report: It took 500 milliseconds to rotate 90 degrees counterclockwise

```
{
   "cmd":"report",
   "model":"sensor_cube.aqgl01",
   "sid":"158d0000123456",
   "params":[{“cube_status”:”rotate”},{"rotate_degree":-90},{"detect_time ":500}]
}
```

Other status report:

```
{
    "cmd":"report",
    "model":"sensor_cube.aqgl01",
    "sid":"158d0000123456",
    "params":[{"cube_status":"flip90"}]
}
```

