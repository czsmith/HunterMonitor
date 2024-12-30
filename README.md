# HunterMonitor
ESPHome project to monitor Hunter X-Core 400 irrigation controllers and allow remote operation


## Background
Monitoring an irrigation system which is usually programmed to operate during sleeping hours is a bit of a problem. ESPHome seems like a good basis to build a device to monitor the happenings in my Hunter X-Core 400 controller.

It is pretty easy to monitor the power to operate the valves, but finding a way to turn them on and off remotely posed more of a challenge.  An earlier Arduino project by XXX which implemented the Hunter Roam protocol to turn zones on and off looked like a grest start, but it isn't integrated with ESPHome.  Adding the logic to monitor the zones seems possible, but there's a lot of duplication with what ESPHome provides.

Javier Vidao surprisingly posted a component for ESPHome which provides just that control.

This project integrated Javier's work with additional logic to monitor and operate the Hunter X-Core.  

## Functionality
Functionality includes:
- Monitor the operation of up to four zones.  (OK, actually the voltage to drive the valves that water the zone.)
- Allow individual zones to be turned on and off remotely.  (Just one at a time as this is an X-Core limit)
- Allow a program to be started and stopped remotely.
- Suppress irrigation, even when called for by an existing program.
- Power the monitor from the existing X-Core power supply. (No USB needed.)
- Provide a local web UI
- Integrate with HomeAssistant
- Package the unit to be small enough to fit inside the X-Core controller housing.

## Schematic
The monitor uses an ESP8266 running ESPHome.
![Schematic_Hunter_2024-12-30](https://github.com/user-attachments/assets/7e0c6747-26bc-4b7a-902d-c6eef75c3813)

### Comments: Zone Monitoring
The four zones are monitored by determinig if power is provided to the corresponding zone.  Power is 24-28VAC so AC opto-couplesrs are used to convert the signal to one appropriate for the ESP8266.  

The output of the opto-coupler is a 120Hz pulsing signal as there is no hardware filtering.  ESPHome filters the signal in software.

Also, came up one digital GPIO short (all the others have some restriction that messes up booting) so I used the analog input, A0, and some ESPHome filtering to derive a digital signal.

### Comments: Controlling the X-Core
The Hunter X-Core models support the Hunter ROAM protocol.  This is a one-way mechanism to signal to the controller to start or stop a zone or to run a program. In brief, there is a negative voltage between the REM terminal and the right 24VAC terminal on the controller.  A proprietary protocol sends a message by pulling down the REM terminal to send start, stop and data signals.  

Other people have worked this problem and developed code and solutions to controlling the X-Core. Notably, encodina (https://github.com/ecodina/hunter-wifi) and javiervidao (https://github.com/javiervidao/Hunter_Roam).  A long-running discussion on the HomeAssistant blog site covers a lot of this material, https://community.home-assistant.io/t/irrigation-hunter-x-core-remote-control-using-rem-pin/320786.

An opto-isolator (4n35) is used here to allow the ESP8266 to drive the negative signal needed as well as provide general isolation from the X-Core.  This project uses Javier Vidao's hunter-roam ESPHome component.

### Comments: Suppressing Irrigation
The X-Core is normally programmed to operate on its own.  To stop this at home, I can just go and turn the main dial to the OFF setting.  Remotely, I could wait for a zone to start watering and then command it to stop.  A better way is to simulate the rain sensor.  Normally, the two SEN terminals are shorted together.  The real rain sensor is a normally closed swithch that is opened when enough rain has fallen. Assuming the "Sensor Bypass" switch is in the "Active" position, we merely need to open the circuit between the two SEN terminals.  Again, an opto-isolator normally keeps the circuit shorted to enable operation.  By opening the circuit, operation of the controller stops.

### Comments: Powering the Unit
Ideally, the monitor unit should be powered from the irrigation controller itself, as it provides readily-accessible 24VAC.  (More like 27 or 28VAC). It is a bit of a stretch to reduce this to the 3.3VDC that the ESP8266 wants.  A small transformer would work, but they're a bit bulky. This project opts to use a DC-DC buck converter.  The 24VAC is put through a full bridge rectifier and filtered with a 10uF capacitor.  This feeds the buck converter.

One problem is finding a converter which will take this high an input voltage. Normally, an LM2596 would work great, but they're pretty large. I opted to use a small V1426, readily found on amazon or aliexpress.  (This is pushing the limits in this desigh, so I expect to add some additional protective circuitry or voltage reduction before committing to a PCB design.)

The schematic allows using either a 3.3 or 5V converter by setting the proper jumper at J2.  If the boad is to be powered via USB, then leave J2 unjumpered. And you can remove all the power supply components as well: R8, D1, C1, U7 and J2.

### Comments: Zone Valve Common Signal Source
Normally, zone valves are wired between the "C" termain and the particular zone terminal.  From experimenting, I believe the "C" terminal is the same as the right 24VAC terminal, but I'm not sure.  To be safe, the board can be wired to both 24VAC-R and "C" while leaving J1 unjumpered.  If you do not care to wire "C" to P2, then jumper J1 and the 24VAC terminal will be used as the common for sensing zone valve activation.

### Comments: Wiring
The symbols should be pretty obvious. Just make sure that you the the Right/Left correct for the 24VAC and SEN terminals. If you have three zones or fewer, you can get everything on a piece of CAT5 cable. (24VAC R+L, REM, SEN R+L, and Z1-3).

## ESPHome 
This is a standard ESPHome project.  Copy and modify the "hunter-roam-alt.yaml" file for your environment, compile and download it.

Signing onto the local UI, we get:
<img width="2400" alt="ScreenShot" src="https://github.com/user-attachments/assets/fbc2d18d-25fd-4012-b2c4-091f1a8889e6" />

### Comment: Zone Filtering
The actual signal coming out of the opto-isolators that monitor the zones are not clean.  There's a pulse 120 times a second as the AC signal is near zero. The signal goes OFF briefly. To ignore this, a filter says the signal must be off for a minumum of 100msec.  This causes all the temporary pulses to be ignored. 

For Zone 4 which comes in on the analog signal, we have to pick a sampling interval.  It's possible that we'd smaple the signal during a pulse, os we ask for the minimum of a number of samples that are taken at something other than a multiple of 120 samples/second.  Of course, this is originally representated as a number from 0 to 1V.  We multiply it by 3.3 to get the real input voltage and choose an ON threshold as something over 2.5V.

## HomeAssistant
Once the ESPHome project is working, it is easily integrated into HomeAssistant using the ESPHome integration.

When loaded, you'll have the following elements to play with:
<img width="1115" alt="HomeAssistantConfiguration" src="https://github.com/user-attachments/assets/a4583201-780c-4a58-a64f-8c8d07c496f1" />
