# newmotion-p1-spoofer
Automated dynamic load balancing and solar surplus charging for legacy NewMotion Home Advanced 2.2 EV chargers by injecting spoofed P1/DSMR telegrams via a Waveshare ESP32-S3-RS485-CAN control board.

# Spoofing signal sent to NewMotion charger through P1

Unfortunately, I have an older version of the NewMotion charger which doesn’t support Modbus over TCP, so I decided to spoof fake P1 messages myself.
This enables me to control the charging speed to either only use solar surplus, or to make sure my consumption stays under the max configurable power limit for the capacity tariff.
I am abusing the loadbalancing feature from the charger to control the charging power of the car.

My logic is for a 3x230V without N, a **NewMotion Home Advanced 2.2** charger connected in monophase and a max house consumption of 32A.
You might need to adapt the calculations if you have a different electricity connection or another max house consumption.

My **NewMotion Home Advanced 2.2** charger has load balancing with 1 CT clamp. The original NewMotion P1 device sends P1 messages with the measured current each 10 seconds to the charger.
In fact, in the case of a charger connected in monophase on 3x230V without neuter, the loadbalancing from newmotion is incorrect as it only measures the current of 1 phase, and there might be a higher current on the other phase.
I only allow the total current over the 3 phases to be max 6000W => this is lower than the max between 2 phases: 230V*32A=7360W.
So this is safer than the loadbalancing from Newmotion whoch only checks one phase.

<img width="108" height="259" alt="image" src="https://github.com/user-attachments/assets/fa5a6914-a3a6-42fe-af8e-d4a6f14869c8" />

<img width="178" height="238" alt="image" src="https://github.com/user-attachments/assets/868372e5-ebcb-4575-a3d9-0756744b52be" />

The RJ11 connector of the newmotion P1 device has the following order of colours in my installation:
white orange, orange, white green, blue, white blue, green

## 🔍 How to Sniff the P1 Messages

I added a parallel RJ11 splitter in between (Twenty4seven P1 Splitter: https://www.bol.com/be/nl/p/twenty4seven-p1-splitter-6-poort/9300000222709642/) and connected a P1 RJ11-to-USB reader (https://www.bol.com/be/nl/p/twenty4seven-slimme-meter-kabel-usb-en-rj11-wifi-p1-meter-1-8-meter-p1-splitter-home-assistant-raspberry-pi-4/9300000290115507/). 

See some examples of raw messages in `sniffedP1messages.txt`.

The P1 messages contain a specific firmware version; it might be different for your charger. It is best to sniff with a P1 reader first to make sure the P1 message layout matches. 
You might need to change the firmware version string inside your spoofing scripts to match your unit.

---

## 🛠️ Hardware Setup

I bought the following device to replace the original NewMotion device:
* **Waveshare ESP32-S3 RS485 CAN**   https://www.amazon.com.be/-/nl/dp/B0FMYL5SZX?ref=ppx_yo2ov_dt_b_fed_asin_title

### Waveshare Board Header Layout
If you open the device case, you can see the black header block with these pins:

<img width="435" height="339" alt="image" src="https://github.com/user-attachments/assets/2758b60c-156e-43cc-8986-662c4380d94f" />

<img width="482" height="225" alt="image" src="https://github.com/user-attachments/assets/4c7f48b8-3a9f-4460-b536-5c8c050a9b9c" />

Only the **blue** and **yellow** dupont wires are used. The other wires visible in the picture were there for testing and also help keep the connections securely attached.

Hailege Jumper Wires 200 stuks/5 x 40 Pin Dupont Wire Assortiment Kit voor Protype Board  https://www.amazon.com.be/-/en/Hailege-Jumper-5x40Pin-Assortment-Protype/dp/B0BN1LJ843
You only need 2 2,0 mm dupont wires , but this pack was cheaper ;-) 

---

## 🪛 Physical Wiring (Wago Splicing Hack)

I cut the wires of the RJ11 connector of the NewMotion P1 device (leave enough space for connecting them again afterwards if needed) and bridged them using Wago connectors. Only **4 wires** are actually used: `orange`, `orange-white`, `green-white`, and `blue-white`.

### 1) Orange-White / Orange: Handshake
* **Orange** = 5V continuous. 
* Orange connects internally to the P1 LED on the board and returns the current over the **orange-white** line.
* The charger expects to see this current flowing back on orange-white, or it refuses to charge.
* **Current setup:** I left these attached to the wires going to the RJ11 connector of the original NewMotion P1 device so it still passes through the P1 LED.
* *Next step / Future improvement:* This can probably be replaced with a simple `1kΩ` resistor between the orange and orange-white wires.

* **Wago 1:** Orange wire coming from the CAT7 cable from the charger + Orange wire going to the RJ11 connector of the NewMotion P1 device.
* **Wago 2:** Orange-white wire coming from the CAT7 cable from the charger + Orange-white wire going to the RJ11 connector of the NewMotion P1 device.

### 2) Green-White: Ground
* **Wago 3:** Green-white wire coming from the CAT7 cable from the charger + Green-white wire going to the RJ11 connector of the NewMotion P1 device + **Blue Dupont wire** which is connected to **PIN 3 (GND)** of the Waveshare board.
* *Note:* Once I have the 1kΩ resistor installed between the orange and orange-white lines, I should be able to get rid of the white-green line to the NewMotion device as well (Still to be tested).

### 3) Blue-White: P1 Data Send
* **Wago 4:** Blue-white wire coming from the CAT7 cable from the charger + **Yellow Dupont wire** which is connected to **PIN 15 (GPIO6)** of the Waveshare board.

*See `spoofnewmotion.yaml` for the code to spoof the P1 message every 10 seconds. I also retrieve data from my HomeWizard P1 meter every second, as Loxone only allows polling every 10 seconds.*

---

## 🧮 Logic to Control the Charger

In my case, my main home electrical installation can handle a maximum of **32A**. 
All future logic is assuming a 32A max houseload, you might need to change it to your max current.

You can monitor the live charger status by querying its local IP: `http://<IP_OF_CHARGER>:12800/user/status`

### Status Payload Example:
```json
{
  "cpid": "cpid",
  "serial": "serial",
  "model": "HOMEADVANCED",
  "imsi": "",
  "iccid": "iccid",
  "meterSerial": "meterserial",
  "meterType": "3f_Inepro",
  "ocppType": "JOCPP16",
  "ocppEndpoint": "wss://://evc-net.com",
  "ocppProtocolType": "ocpp/json",
  "ocppStatus": "Accepted",
  "modemStatus": "Modem not active with csq +CSQ: 8,99 not connected ip: ip_address",
  "connectors": [
    {
      "id": 1,
      "type": "Type2 Cable",
      "status": "ON",
      "max": {
        "phases": "PHASE1",
        "current": 25,
        "currentArray": [25, 0, 0]
      },
      "plugMax": {
        "phases": "PHASE3",
        "current": 53,
        "currentArray": [53, 53, 53]
      },
      "limit": {
        "phases": "PHASE1",
        "current": 14.1,
        "currents": [14.1, 0, 0],
        "currentArray": [14.1, 0, 0]
      },
      "chargingRate": "Charging rate: 0kw 3 phase [0A pf ] ",
      "chargingNeed": "",
      "errors": [],
      "warnings": []
    }
  ],
  "maxLimit": {
    "phases": "PHASE3",
    "current": 32,
    "currentArray": [32, 32, 32]
  },
  "status": "ON",
  "version": "1.13.1.3",
  "errors": [],
  "warnings": []
}
```

### Headroom Calculations
The charger uses the maximum **32A** threshold to determine the new allowed charging current:
* If the spoofed P1 message contains **30A**, it means: *You can increase the charging current by 2A.*
* If the spoofed P1 message contains **35A**, it means: *You are 3A over the max allowed house current. Lower the charging current by 3A.*

### Control Variables
* I have a switch named `Solar only`.
* I have a slider for max house consumption: `capacity_limit_kw`.
* **If Solar Only = ON:** `max_allowed_consumption` is set to **0 kW**.
* **If Solar Only = OFF:** `max_allowed_consumption` is set to `capacity_limit_kw`.

```text
esp32 P1 Active Power kW = Amount of power being consumed now over all phases.
Negative value = Exporting to the grid
Positive value = Importing from the grid

spoofed_current = INT(32 + ((max_allowed_consumption_kw * 1000) / 230))
(32 = Max current for my house installation)
```

### Safety Limiter
The current charging power is obtained from: `http://<IP_OF_CHARGER>:12800/user/status` via the string block: `\i"chargingRate":"Charging rate: \i\v`.

```text
charging_current = charging_power / 230
```
On a monophase setup (my case), the car needs to be charged with **at least 6A**. 
```text
Future charging current = charging_current + (32 - spoofed_current) > 6
Therefore: spoofed_current < charging_current + 32 - 6
```
**Important:** Add a limiter constraint in your logic so that `spoofed_current` is never allowed to exceed this value!

---

## 💻  Modbus Settings
This limited `spoofed_current` value is sent over Modbus TCP to the Waveshare board:
* **IP Address:** `<IP_OF_WAVESHARE>:502`
* **Device Address:** `1`

### Modbus Actuator Configuration Layout:
* **IO Address:** `1`
* **Command:** `6 - Write Single Register`
* **Data Type:** `16-bit Unsigned Integer`
