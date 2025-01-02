## Concept
This is a 3D printer Toolead board intended to print in a "high temperature" exclosure. 80C is the target limit. This is hot for consumer pastics (Nylon, PA) but not hot for industrial plastics (PEEK & PEI - 90  to 140C). This should provide good conditions for printing the new PPA based filaments.

Because of the chamber temperature the 2 main parts that are most susceptible to heat are kept out of the chamber: the heater MOSFET and the stepper driver. The board features passthrough ports for these items to make connections easier.

## Features

* 1 x 6 pin CLIK-Mate connector to bring in USB data, 12-24V power and your choice of fan voltage.
* 1 x 4 pin port for full bridge load cells using the ADS1220 ADC sensor
* 2 x 3 pin Fan ports with RPM monitoring (Heatbreak Cooler & Part Cooler). Max 2A power draw for the fans, limited by the input pin.
* 1 x 2 pin PTC100 port for Hot End temperature using the MAX31865 RTD Sensor (can alternatly be built for PT1000)
* 1 x 2 pin NTC thermistor port for Heatbreak temperature monitoring
* 1 x 4 pin Neopixel/Dotstar port supplied with 5V/500mA for toolhead lighting
* On board NTC thermistor for board temperature monitoring
* On board power conversion from 12V to 5V and 3.3V using a buck converter module and two LDOs for ADC reference level DC power supply. (Does not reply on your 5v or 3.3v source being noise free)
* USB communication via wire pair in the main connector (no separate USB connector)
* STM32 F103 Core M3 CPU @72Mhz
* Singe wire debug port for flashing the MCU with Katapult
* Reset button
* MCU Boot indicator LED - also indicates USB full speed pull-up enabled
* Molex CLIK-Mate connectors. Support up to 2A of current and are much nicer to use vs JST alternatives.
* ESD protection on all I/O pins.
* Guard Ring for ESD protection and grounding to chassis

## Squaks
This section is a TODO list of issues and subjects not yet thought through:
* Add registration holes to the stencil layer
* Consider re-adding the fan LEDs - not convinced they will be useful.
* Carefully double check the pin assignments on the MCU for all pins. PWM, ADC, SPI etc.

## References
You cant learn anything quickly without looking at the work of others. I specifically want to credit:
* [HUVUD](https://github.com/bondus/KlipperToolboard) - This is a toolhead board with an integrated stepper drive that fits on the back of a NEMA17 stepper motor. This has some incredibly tight packaging and routing in the PCB design. This particularly encouraged me to stop worrying about overlapping cortyards and just pack things close together.
* [Prusa's Open Source](https://www.prusa3d.com/page/open-source-at-prusa-research_236812/) efforts and specifically the Love Board ([schematics](https://www.prusa3d.com/downloads/Electronics_drawings/MK4_electronics_schematics.zip)) - This is the first commercial open source board with a load cell ADC. It also pays particular attention to ESD supression and noise filtering which I found absent in other projects. The level of robustness they are achieving is worth our attention.
* The Voron project's [Klipper Expander](https://github.com/VoronDesign/Voron-Hardware/tree/master/Klipper_Expander) - this was useful for getting the Fan MOSFETs correct.
* [Tiny Fan Board](https://github.com/Gi7mo/TinyFan) -  I was particularly interested in having RPM monitoring on the fan ports and this board implements that.
* Rick Heartly for this [video on grounding](https://www.youtube.com/live/ySuUZEjARPY) and PCB stackups. I'm using his preferred 4 layer stackup for this board and implementing his ideas on power planes and via stitching.

## Detailed Design Topics
Everything, in no particular order...

### No USB Port
USB is used as the communications protocol. A USB port was not used because it consumed valuable board space. Also, USB-C ports feature some very small pads that I didnt want to hand assemble. The USB D+ and D- wire pair must be twisted together to provide a good connection. A shielded twisted pair cable could also be used for these wires.

### Power

General Power Architecture:

12V / 24V --> Buck Converter --> 5.75V |--> Precision LDO --> 5V
                                       |--> Precision LDO --> 3.3V

To get the most out of the ADS1220 ADC a very low noise 5V reference is required. Typical LDOs are good for this but there is a more specialized class of LDO for measurment applications like this 24bit ADC. Thats what I am using here. These LDOs are voltage spcific so no external tuning resistors are required. They also have built-in reverse voltage protection so the debug port can power the board without the need for extra parts.

#### LDO Selection

Features considered when selecting the LDO:
* Support 500mA of current output
* Fixed Voltage - eliminate external resistor netowork and saves space
* Reverse voltage protection - eliminate external diodes
* Low output voltage noise, less than 25μVRMS
* A package large enough to hand assemble
* Low RθJA - Junction-to-ambient thermal resistance

I've selected this part: TI [TPS7A21-Q1](https://www.ti.com/product/TPS7A21-Q1)

The last point about RθJA requires some math to validate. Basically this is the value that will end up limiting the current the device can supply when its inside a hot chamber at 80C.

```
SUPPLY_CURRENT = ((T(JUNCTION_MAX) – T(APPLICATION_TARGET)) / RθJA) / (V(SUPPLY) - V(OUTPUT))
```

For the TPS7A21-Q1, the T(JUNCTION_MAX) is 150°C, the RθJA of the VSON package is 58.9°C/W. The maximum input voltage to the LDO is 6V, but the target input voltage is 5.5V. Results with a value larger than the devices capacity of 500mA means there is some headroom.

```
3.3V Current @ 80°C = ((150°C - 80°C) / 58.9°C/W) / (5.5 - 3.3) = (1.19) / 2.7 = 540mA
5V Current @ 80°C   = ((150°C - 80°C) / 58.9°C/W) / (5.5 - 5.0) = (1.19) / 1.0 = 2376mA
```

If you run the calculations for a 100°C target chamber temp you still get results that are compleatly usable:

```
3.3V Current @ 100°C = ((150°C - 100°C) / 58.9°C/W) / (6.0 - 3.3) = (0.85) / 2.7 = 358mA (thermally limited!)
5V Current @ 100°C   = ((150°C - 100°C) / 58.9°C/W) / (6.0 - 5.0) = (0.85) / 1.0 = 1697mA
```

And the good news is the 5V LDO still has headroom and it is the one running the Neopixels which will pull the majority of the current from. The 3.3V rail shouldn't be pulling more than 200ma.

It is important to note that only the "DRB" 3mm x 3mm package has the good RθJA. The Specific part numbers are:

TPS7A2150PQWDRBRQ1 - 5V
TPS7A2133PQWDRBRQ1 - 3.3V

#### Buck Converter Selection
This components role is to steps down the voltage from 24V to 5.5V without generating lots of heat.

Features considered when selecting the Buck Converter:
* A Module with integrarated inductor - no hand layout required
* Support at least 1A of current output to drive both LDOs
* Support 24V input voltage which is common on 3D Printers
* A package large enough to hand assemble
* Low RθJA - Junction-to-ambient thermal resistance

The Buck converter is a module with a built in inductor. There is a whole art to using buck converters with external inductors. The routing of the traces can produce noise if not done correctly. A module sidesteps these concerns and reduces the complexity and part count. It just needs bulk capacitors and a resistor feedback network to opperate.

TI parts TPSM336x5 and TLVM236x5, which are the same architecture with different ranges of ouput voltage selection. Since the LDO's can accept no more than 6V, the [TLVM23615](https://www.ti.com/lit/ds/symlink/tlvm23615.pdf) is a good fit as it gives better resolution (more resistor choices) in voltage selection to 5.5V.

Thermal calculations for the buck convter are a bit different.
* T(JUNCTION_MAX) is 125°C
* Approximate efficiency is 87% or 0.87
* RθJA = 22°C/W

```
IOUT MAX = ((T(JUNCTION_MAX) – T(APPLICATION_TARGET) / RθJA) * (EFFICIENCY / 1 - EFFICIENCY) * (1 / VOUT)
```

Again, any result greater than the devices rated current of 1.5A is an indication of headroom:

```
IOUT MAX @ 80°C = ((125°C - 80°C / 22°C/W) * (0.87 / (1 - 0.87)) * (1 / 5.75) = 2.4 A
IOUT MAX @ 100°C = ((125°C - 100°C) / 22°C/W) * (0.87 / (1 - 0.87)) * (1 / 5.75) =  1.3 A (thermally limited!)
```

The module is not thermally limited at the target temp of 80C and has enough headroom to meet the applicaiton requirements at 100C.

##### Resistor Selection
TLVM23615 uses a resistor divider to set the output voltage between 1V and 6V. The LDO's require some margin above 5V for dropout and for voltage sags so a target of 5.5V was selected. The two resistors are called RFBB and RFBT. RFBB is typically 10kΩ and RFBT is calculated as:

RFBT kΩ = RFBB kΩ × (VOUT - 1)
RFBT kΩ = 10kΩ × (5.5 - 1) = 45kΩ

Unfortunatly, 45kΩ isn't a common size. But 45.3kΩ is. So what voltage will that yield:
45.3kΩ = 10kΩ × (VOUT - 1)
(45.3kΩ / 10kΩ) = VOUT - 1
(45.3kΩ / 10kΩ) + 1 = VOUT
VOUT = 5.53V

This is prefectly acceptable.

### Stackup & Ground Planes
The PCB stackup is: Ground, Signal/Ground, Power/Signal, Ground.

Layer 1 - Components and Ground. Wherever practical, traces on this layer drop into a via and remain as short as possible.
Layer 2 - Primary signal routing plane. Any long traces that would have cut thr power plane are routed on this layer.  Some care is taken to route traces under ground on the layer above but this wasnt practical everywhere.
Layer 3 - Power plane + secondary signal layer. 3.3v, 5V and 12V power is poured in large planes on this layer. The board components are organized such that 5V and 12V parts are grouped on different edges of the board so the power planes dont need to overlap. Singals routed on this layer need extra care not to cut or choke the power zones.
Layer 4 - Ground plane for layer 3

The LDO's and Buck Converter get their best thermal performance when the board has as much copper in it as possible. Basically its a large heat spreader. Since the plan is already to pour ground everywhere this fits perfectly with what the VRMs want.

### Guard Ring
The bord is circled by a guard ring that is 2mm wide, placed on every layer of the board and stitched together with vias. The copper is exposed to allow it to be condutive. The mounting holes and mounting points on the external connectors are all tied into the guard ring.

A spot for a TVS diode to bridge the guard ring to the ground plane is provided but at the time of design its not clear if this is the right thing to do. Other options exist:
* GND and he guard ring can be connected with a large resistor.
* The guard ring can be left floating to pass ESD spikes around the board and back into the printers frame via the mounting holes.
* The load cell can be grounded to the guard ring by attacing a ring terminal to a ground wire and fixing it to the adjacent mounting point.
* The guard ring can be connected back to chassis ground via a dedicated wire back to the electronics bay. Again this can be done with ring terminals.
* ESD transmissable plastic could be used to create a cover for the board that would then be electrically connected to the guard ring.

#### Via Stitching
Ground planes on layers 1, 2 and 4 and the Guard Ring have via stitching applied.

### ESD Protection
ESD protection, we should have some.. ESD protection devices on digital lines semms to be industry standard practice and Prusa's design seemes to prioritize this as well.

Large TVS diode between the main 24V supply and the ground plane to stop power entry ESD.

Unidirectional 3.3v TVS diodes on:
  * ADC input line for the Heatbreak Thermistor
  * ADC input lines for the Fan RPM
  * ADC input for the filament sensor (filament sesnor can be on/off use an ADC value)

Bi-direction 3.3V TVS diodes on:
  * USB Pair

Unidirectional 5v TVS diodes on:
  * LED clock and data output line

### High Frequency Noise Filtering
Sensitive components are guarded with Ferrite beads (120R@100Mhz/3A) to filter high frequency noise.
* Load Cell lines
* Filament sensor power lines
* PT100 port lines
* Heatbreak thermistor lines
* LED power lines

### Trace Widths
The board isnt "Controlled Impedance" but im using the trace widths from the data table from the board house: [OSH Park 4 Layer Impedance Table](https://docs.oshpark.com/resources/four-layer-impedence-table.png)

| Track Width  | Usecase |
|--------------|---------|
| 9mil         | 90 Ohm Impedance lines for USB interface |
| 15mil        | 50 Ohm Impedance lines for SPI, ADC & PWM lines |
| 0.3mm, 0.4mm | Pin width of most of the ICs. Used for power decoupling capacitor links  |

### Via Sizing
The default via size of 0.3mm / 0.6mm have an ampacity of over 2A. Thats good for basically everything on the board.

On the USB lines a minimum sized via will impact the rise/fall times less.

### Capacitor Sizes
Capacitors de-rate at the target 80C chamber temperatures. Larger capacitors de-rate less than smaller ones. I'm also hand assembling the board and decided the smallest component I want to place manually is a 0603. This chart is used to select capacitor sizing:


| Range C | SMD Size | Type   | Voltage | Range |
|---------|----------|--------|---------|-------|
| 1pf - 10nf | 0603 | Ceramic C0G | 25V | 10% |
| 10nf - 0.1uf | 0603 | Ceramic X7R | 25V | 10% |
| 0.1uf - 2.2uF | 0805 | Ceramic X7R | 25V | 10% |
| 2.2uF - 22uF | 1206 | Ceramic X7R | 25V | 10% |

Nothing in the design requires an electrolytic capacitor.

## Power

## LEDs
Leds are driven by 24V, 5V and 3.3V power. Resistors are selected to limit the LED's driven current to about than 2ma. 3.3V output pins on the MCU supply a maximum of ~2.0mA current.

LIST-C193KRKT-5A - LiteOn
Forward Voltage: 2.0V
Forward Current: 1.5mA

| LED Module | Supply Voltage | Resistor | Current | Forward Voltage (FV) |
|------------|----------------|----------|---------|----------------------|
| LIST-C193KRKT-5A | 24V  | 10KΩ | 2.2mA | 2.0v |
| LIST-C193KRKT-5A | 5V   | 2kΩ | 1.5mA | 2.0v |
| LIST-C193KRKT-5A | 3.3V | 1kΩ | 1.3mA | 2.0v |

Resistors should be able to handle 1/2 watt of power or better. Usually this requires 0805 size resistors.

### Fan Voltage
An inout pin allows you to pick the supply voltage for the fans. I dont have to generate 12V on board to run them.

#### Fan RPM Monitoring
My personal view is that hot end cooling fans are safety critical systems that should be monitored. If the fan fails the printer should shut down the heater. So the fans on this board feature an RPM monitoring pin. There are plenty of fans in the supply chain that have this feature.

#### Fan PWM Output
4 Wire fans in this size do exist but they are very rare. If you use a 4 wire fan, you dont need to put MOSFETs on the board to drive the fans. This effectivly means you can get the MOSFET into the place where its more likely to get cooled. But the lack of fans means this is very unlikely to happen, so I left this out. If i had my way I would require 4 wire fans and omit the MOSFETs.

#### Tach Line Resistor
Fans report their RPM or Tachometer as a square wave. An ADC reference voltage is supplied in to the fan's tachometer input. The fan then pulls this line low to create a square wave. This means the tach line voltage supply connects to ground inside the fan. Both [Noctua](https://noctua.at/pub/media/wysiwyg/Noctua_PWM_specifications_white_paper.pdf) and [Sunon](https://www.sunon.com/PROSEARCH_FILES/(D04115420G-01)-1.pdf) require limiting the current flowing through the tach line to not more thean 5mA. This is important to do as, when the RPM is 0, the line may be pulled low continouously causing a large current flow through the fan which may cause it to overheat. All the other designs I have seen omit this resistor.

E.g. the [Tiny Fan Board](https://github.com/Gi7mo/TinyFan) uses an in-mcu pull-up on the ADC line and connects it directly to the Tachometer input. The Pins on an STM32 chip can supply up to 25mA which is 5x higher than the industry spec.

```
R = V/I
R = 3.3V / 5mA
R = 660 Ohm
```

This is really close to 1k, so thats what im using as this helps with BOM consolidation.

### Fan MOSFET Selection
Largely this design is copied. The MOSFET selected needs to:
* Support 24V of supply
* Work with 3.3V logic to trigger the gate

The devices I have seem are rated for 3A to 5A of current at 30V. This is hugely over rated for the application, but I suspect this helps keep them cool. Or it has something to do with large currents required to start the fan.

I increased the gate resistor value from 100 Ohms to 1K Ohms. Some designs omit this resistor, but this increases power used by the MOSFET and drives up its temperature.

I'd like to have some math to work out the how fast the gate needs to switch and calulate an optimal value. Klipper's default switching time is 10ms in software PWM mode. This is 100Hz, a very slow switching speed. It is possible that higher switching speed might work for some fans when using hardware PWM.

### Crystal

I opted to use an Oscillator module instead of a crystal + capcitors. The module combines a crystal with the correct capacitors and a compact footprint. Capacitor tuning is something that requires more sensitive equipment than I haveto validate the frequency produced by the circuit. Similar to my approach to power, I wanted to avoid anything that introduced complexity and a point of failure.
