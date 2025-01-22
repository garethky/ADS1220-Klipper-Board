## Concept
This is a 3D printer Toolead board intended to print in a "high temperature" exclosure. 80C is the target limit. This is hot for consumer pastics (Nylon, PA) but not hot for industrial plastics (PEEK & PEI - 90  to 140C). This should provide good conditions for printing the new PPA based filaments.

Because of the chamber temperature the 2 main parts that are most susceptible to heat are kept out of the chamber: the heater MOSFET and the stepper driver.

## Features

* 1 x 6 pin CLIK-Mate connector to bring in USB data, 12-24V power and your choice of fan voltage.
* 1 x 4 pin port for full bridge load cells using the ADS1220 ADC sensor
* 2 x 3 pin Fan ports with RPM monitoring (Heatbreak Cooler & Part Cooler). Max 2A power draw for the fans, limited by the input pin.
* 1 x 2 pin PTC100 port for Hot End temperature using the MAX31865 RTD Sensor (can alternatly be built for PT1000)
* 1 x 2 pin NTC thermistor port for Heatbreak temperature monitoring
* 1 x 4 pin Neopixel/Dotstar port for illuminating the print are supplied with 5V/500mA for toolhead lighting
* On board NTC thermistor for board/ambient temperature monitoring
* On board power conversion from 12V to 5V and 3.3V using a buck converter module and two LDOs for ADC reference level DC power supply. (Does not reley on your 5V or 3.3V source being noise free)
* USB communication via wire pair in the main connector (no separate USB connector)
* STM32 F103 Core M3 CPU @72Mhz
* Singe wire debug port for flashing the MCU with [Katapult](https://github.com/Arksine/katapult)
* Reset button
* MCU Boot indicator LED - also indicates USB full speed pull-up enabled
* Activity LEDs for both fans, 5V and 3.3V power
* Molex CLIK-Mate connectors. Support up to 2A of current and are much nicer to use vs JST alternatives.
* ESD protection on all I/O pins.
* Guard Ring for ESD protection and grounding to chassis

## Squaks / TODO
This section is a TODO list of issues and subjects not yet thought through:
* Double/Tripple check the pin assignments on the MCU for all pins. PWM, ADC, SPI etc.

## References
You cant learn anything quickly without looking at the work of others. I specifically want to credit:
* [HUVUD](https://github.com/bondus/KlipperToolboard) - This is a toolhead board with an integrated stepper drive that fits on the back of a NEMA17 stepper motor. This has some incredibly tight packaging and routing in the PCB design. This particularly encouraged me to stop worrying about overlapping cortyards and just pack things closer together.
* [Prusa's Open Source](https://www.prusa3d.com/page/open-source-at-prusa-research_236812/) efforts and specifically the Love Board ([schematics](https://www.prusa3d.com/downloads/Electronics_drawings/MK4_electronics_schematics.zip)) - This is the first commercial open source board with a load cell ADC. It also pays particular attention to ESD supression and noise filtering which I found absent in other projects. The level of robustness they are achieving is worth our attention.
* The Voron project's [Klipper Expander](https://github.com/VoronDesign/Voron-Hardware/tree/master/Klipper_Expander) - this was useful for getting the Fan MOSFETs correct.
* [Tiny Fan Board](https://github.com/Gi7mo/TinyFan) - I was particularly interested in having RPM monitoring on the fan ports and this board implements that.
* Rick Heartly for this [video on grounding](https://www.youtube.com/live/ySuUZEjARPY) and PCB stackups. I'm using his preferred 4 layer stackup for this board and implementing his ideas on ground planes and via stitching.

## Detailed Design Topics
Everything, in no particular order...

### Optional USB Port
USB is used as the communications protocol. The USB D+ and D- wire pair must be twisted together to provide a good connection. A shielded twisted pair cable could also be used for these wires, so long as it can meet the requirements for continuous flexing and chamber temperatures.

I didnt see a comprelling reason to opt for CAN bus. It requires extra hardware on the host. The cable requirements for CAN are similar to USB. (remember we aret doing the super high speed version of USB)

A USBC port is included on the back side of the part and not intended to be used during normal opperation. It is included for bench running the part and performing firmware updates before it is installed into a printer. Inital flashing can be accomplished via the debug port, so this hardware isnt required. USB cabes are not rated for continous flexing conditions and high temperatures in a 3D printer. We usually build custom cables out of PTFE coated wire that can flex, resist abrasion and withstand the high temps.

The power supplied by the USB cable runs the MCU. A protection diode is incuded to stop power flowing back over the USB cable if the main power supply and the USB supple are connected at the same time. Due to voltage drop at the protection diode the 5V LDO wont see more like 4.5V and will either run in dropout mode or fail to start. If the 5V section of the board doesnt work under USB power this is OK as its just a debugging connector. If opperation is required for some test, just plug in the 24V/12V supply.

### Power
General Power Architecture:

5V USB Power |
             |
12V / 24V    --> Buck Converter --> 5.75V |--> Precision LDO --> 5V Digital
                                          |--> Precision LDO --> 5V Analog
                                          |--> Precision LDO |
3.3V Debug Power ------------------------------------------- --> 3.3V Digital

To get the most out of the ADS1220 ADC a very low noise 5V Analog reference is required. Typical LDOs are good for this but there is a more specialized class of LDO for measurment applications like this 24bit ADC. Thats what I am using here. These LDOs are voltage spcific so no external tuning resistors are required. They also have built-in reverse voltage protection so the debug port can power the board without the need for extra protection diodes.

* When using the SWD header, 3.3V power is injected directly into the 3.3V power bus
* When using the USB connector 5V power is fed into buck converter via a reverse current protection diode. The buck converter opperate din dropout mode and passes on the 5V to the LDOs. The 5V LODs will alos opperate in dropout mode.
* When bother USB and 24V are connected the protection diode stops 24V from backfeeding onto the USB power line.

#### LDO Selection

Features considered when selecting the LDO:
* Support 500mA of current output
* Fixed Voltage - eliminate external resistor netowork and saves space
* Reverse voltage protection - eliminate external diodes
* Low output voltage noise, less than 25μVRMS
* A package large enough to place by hand
* Low RθJA - Low junction-to-ambient thermal resistance

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
* Support 12V-24V input voltage which are common primary supply voltages on 3D Printers
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

#### Buck Converter Resistor Selection
TLVM23615 uses a resistor divider to set the output voltage between 1V and 6V. The LDO's require some margin above 5V for dropout and for voltage sags so a target of 5.5V was selected. The two resistors are called RFBB and RFBT. RFBB is typically 10kΩ and RFBT is calculated as:

RFBT kΩ = RFBB kΩ × (VOUT - 1)
RFBT kΩ = 10kΩ × (5.5 - 1) = 45kΩ

Unfortunatly, 45kΩ isn't a common size. But 45.3kΩ is. So what voltage will that yield:
45.3kΩ = 10kΩ × (VOUT - 1)
(45.3kΩ / 10kΩ) = VOUT - 1
(45.3kΩ / 10kΩ) + 1 = VOUT
VOUT = 5.53V

This is prefectly acceptable.

### PCB Stackup
The PCB stackup is: Ground/Components, Signal/Ground, Power/Signal/Ground, Ground/Components

OSH Park's 6 layer board is 'built different'. They have one large core in the middle with samller 4mil cores in between the top 3 and bottom 3 layers. This is in contrast to other stackups where its 3 double sided sheets with 2 cores.
```
1) ---Sig/PWR---
2) ---GND---
3) ---Sig/PWR---
   ===Core=== (36mil)
4) ---Sig/GND---
5) ---GND---
6) ---Sig/GND---
```
With this stackup, layers 1 and 3 reference the ground plane on 2. A signal routed on 1 and 3 doesnt require a ground return via because the ground plane doesnt change. Signals on 4 and 6 reference ground on 5. Signals moving from those layers to/from components on 1 need a ground via as a return path next to the signal via.

Because of the large core, there in minimal interaction between layers 3 and 4. But putting power on layer 4 provides a reference plane fore layer 3, in addition to the pure ground plane on layer 2.

The LDO's and Buck Converter get their best thermal performance when the board has as much copper in it as possible. Basically its a large heat spreader. Since the plan is already to pour ground everywhere this fits perfectly with what the VRMs want.

### Guard Ring
The bord is circled by a guard ring that is 2mm wide, placed on every layer of the board and stitched together with vias. The copper is exposed to allow it to be condutive. The mounting holes and mounting points on the external connectors are all tied into the guard ring.

A spot for a TVS diode to bridge the guard ring to the ground plane is provided but at the time of design its not clear if this is the right thing to do. Other options exist:
* GND and the guard ring can be connected with a large resistor.
* The guard ring can be left floating to pass ESD spikes around the board and back into the printers frame via the mounting holes.
* The load cell can be grounded to the guard ring by attacing a ring terminal to a ground wire and fixing it to the adjacent mounting point.
* The guard ring can be connected back to chassis ground via a dedicated wire back to the electronics bay. Again this can be done with ring terminals.
* ESD transmissable plastic could be used to create a cover for the board that would then be electrically connected to the guard ring.

The aim of the prototype is to test these options out in a real printer to see what performs best.

### Via Stitching
Ground planes and the Guard Ring have via stitching applied.

### ESD Protection
ESD protection, we should have some... ESD protection devices on digital lines seems to be industry standard practice and Prusa's design does this as well.

Large TVS diode between the main 24V supply and the ground plane to stop power entry ESD.

Unidirectional 3.3v TVS diodes on:
  * ADC input line for the Heatbreak Thermistor
  * ADC input lines for the Fan RPM
  * ADC input for the filament sensor (filament sesnor can be on/off or use an ADC value)

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
The board isnt "Controlled Impedance" but I'm using the trace widths from the data table from the board house: [6 Layer impedance table](https://docs.oshpark.com/resources/six-layer-impedence-table.png).

I added the trace widths and some custom design rules to route traces by netclass at the appropriate width on outer and inner layers.

### Via Sizing
The default KiCad via size of 0.3mm / 0.6mm have an ampacity of over 2A. Thats good for basically everything on the board. The board house cant go much smaller than this anyway. In some places around the MCU i have used a smaller via size to help make space for the fanout.

### Capacitor Sizes
Capacitors de-rate at the target 80C chamber temperatures. Larger capacitors de-rate less than smaller ones. I'm also hand assembling the board and decided the smallest component I want to place manually is a 0603. This chart is used to select capacitor sizing:

| Range C | SMD Size | Type   | Voltage | Range |
|---------|----------|--------|---------|-------|
| 1pf - 10nf | 0603 | Ceramic C0G | 25V | 10% |
| 10nf - 0.1uf | 0603 | Ceramic X7R | 25V | 10% |
| 0.1uf - 2.2uF | 0805 | Ceramic X7R | 25V | 10% |
| 2.2uF - 22uF | 1206 | Ceramic X7R | 70V | 10% |

Nothing in the design requires an electrolytic capacitor.

### LEDs
Leds are driven by 24V, 5V and 3.3V power. Resistors are selected to limit the LED's driven current to about than 2ma. 3.3V output pins on the MCU supply a maximum of 25mA of current.

[LIST-C193KRKT-5A - LiteOn](https://optoelectronics.liteon.com/upload/download/DS22-2005-077/LTST-C193KRKT-5A.PDF)
Forward Voltage: 2.0V
Forward Current: 1.5mA

| LED Module | Supply Voltage | Resistor | Current | Forward Voltage (FV) |
|------------|----------------|----------|---------|----------------------|
| LIST-C193KRKT-5A | 24V  | 10KΩ | 2.2mA | 2.0v |
| LIST-C193KRKT-5A | 5V   | 2kΩ | 1.5mA | 2.0v |
| LIST-C193KRKT-5A | 3.3V | 1kΩ | 1.3mA | 2.0v |

Resistors should be able to handle 1/2 watt of power or better. Usually this requires 0805 size resistors.

Some tuning of this might be required in the prototype based on brightness.

### Fans

#### Fan Voltage
A dedicated input pin allows you to pick the supply voltage for the fans. I dont have to generate 12V on board to run them.

#### Fan RPM Monitoring
My personal view is that cooling fans are safety critical systems that should be monitored. If the fan fails the printer should shut down the heater. So the fans on this board feature an RPM monitoring pin. There are plenty of fans in the supply chain that have this feature.

#### No Fan PWM Output
4 Wire fans in this size do exist but they are very rare. If you use a 4 wire fan, you dont need to put MOSFETs on the board to drive the fans. This effectivly means you can get the MOSFET into the place where its more likely to get cooled. But the lack of fans means this is very unlikely to happen, so I left this out. If I had my way I would require 4 wire fans and omit the MOSFETs.

#### Tach Line Resistor
Fans report their RPM or Tachometer as a square wave. A reference voltage is supplied in to the fan's tachometer input. The fan then pulls this line low to create a square wave. This means the tach line voltage supply connects to ground inside the fan. Both [Noctua](https://noctua.at/pub/media/wysiwyg/Noctua_PWM_specifications_white_paper.pdf) and [Sunon](https://www.sunon.com/PROSEARCH_FILES/(D04115420G-01)-1.pdf) require limiting the current flowing through the tach line to not more thean 5mA. This is important to do as, when the RPM is 0, the line may be pulled low continouously causing a large current flow through the fan. All the other designs I have seen omit this resistor.

E.g. the [Tiny Fan Board](https://github.com/Gi7mo/TinyFan) uses an in-mcu pull-up on the ADC line and connects it directly to the Tachometer input. The Pins on an STM32 chip can supply up to 25mA which is 5x higher than the industry spec.

```
R = V/I
R = 3.3V / 5mA
R = 660 Ohm
```

This is really close to 1k, so thats what im using as this helps with BOM consolidation.

#### Fan MOSFET Selection
Slection criteria:

* The max voltage is 24V. So at least a 30V (Drain to Source V<sub>DS</sub)) device is required.
* Duty cycle is 100% / full DC on. We do use PWM for the part cooling fan but the hot end fan is usually 100% on.
* Ambiend temp target is 80C.
* The gate drive voltage is 5.0. So V<sub>GS</sub> needs to be less than 3.3V. I'm re-using the level shifter part to step up the 3.3V GPIO voltage to 5V.
* Because of the focus on 100% duty cycle we want a very low resistance MOSFET. Leakage current isnt really an issue in this application.
* The current required is less than 1A. However it has to do this at 100% duty cycle at 80C ambient.

The last part is actually really difficult. Most data sheets show 100% duty cycle under ideal conditions, both gate drive voltage and ambient temperature. We will have neither of these conditions.

I didnt want to conduct a survey of all possible MOSFET parts, so I looked in TI's parts bin and found: [CSD17581Q3A](https://www.ti.com/lit/ds/symlink/csd17581q3a.pdf)

I increased the gate resistor value from 100 Ohms to 1K Ohms. Some designs omit this resistor, but this increases power used by the MOSFET and drives up its temperature. I'd like to have some math to work out the how fast the gate needs to switch and calulate an optimal value. Klipper's default switching time is 10ms in software PWM mode. This is 100Hz, a very slow switching speed. It is possible that higher switching speed might work for some fans when using hardware PWM.

#### Fans and N-Channel MOSFETs
An N channel mosfet switches the ground lead of the fan. This disconnects the fan from ground. This should also defeat the fans ability to pull the Tach line low, disrupting the tach signal. For a safety monitoring usecase, this is an acceptable defect. We dont need the reported RPM to be acurate.

This behaviour is something that I plan to investigate with a scope. I saw some suggestions that the ground lead should bypass the MOSFET with a high value pulldown resistor. I added these as do-not-populate parts on the board to experiment with.

### MCU Crystal

#### Why not an Oscillator Module?
I wanted to use a crystal oscillator module for this project to save on space and complexity. But I found this to be impossible with the current katapult/klipper codebase:

###### OSC_OUT Pin
This is the pin that would normally complete the crystal oscillator circuit. When using an oscillator module its less than clear what must be done from the documentation. But as best as I understand it:
* [AN1709](https://www.st.com/resource/en/application_note/an1709-emc-design-guide-for-stm8-stm32-and-legacy-mcus-stmicroelectronics.pdf) has an explicit diagram with the pin tied to ground when using an external clock source. This fits with the recommendation to tie all unued GPIO pins to GND to minimize noise.
* [AN5286](https://www.st.com/resource/en/application_note/an2586-getting-started-with-stm32f10xxx-hardware-development-stmicroelectronics.pdf) has a similar diagram (Figure 6) that shows the OSC_OUT pin unconnected with a "Hi-Z" note. Hi-Z mean high impedance. There is a "BYPASS" / `HSEBYP` mode that needs to be configured in software that disconnects the OSC_OUTPUT from the clock source. I assume this is how you set the "Hi-Z" mode for the pin.
* I'm pretty sure [this code](https://github.com/Arksine/katapult/blob/25a23cd420d7f0f7b677f1511b5739385fca72d9/lib/stm32f1/system_stm32f1xx.c#L173C12-L173C18) in katapult turns off that bypass mode. This isn't user configurable, so the bypass mode is never available, meaning only crystals should be used.

Its not clear if using the module in non-bypass mode will work correctly if the pin is grounded or floating, since the bypass mode is not engaged.

#### Crystal Capacitor Selection

I selected [ECS-.327-6-34QCS-TR](https://ecsxtal.com/store/pdf/ECX-34Q.pdf) using the STM32 crystal selection tools. I wanted one that had a small footprint and a high temperature rating, this one is good to 125C.

The load capacitance of the crystal is 12.5pF. The 'back of the envelope' method for selecting a capacitor size is to double the load capacitance. So 25pF.

You can do the complete math. But STM's own document, [AN2867](https://www.st.com/resource/en/application_note/an2867-guidelines-for-oscillator-design-on-stm8afals-and-stm32-mcusmpus-stmicroelectronics.pdf), says the best thing to do is to measure the frequency generated with a scope and tune the capacitance based on those results. I'm not sure that my scope is good enough to do this.

### Unused Pins
What is the correct thing to do with unused pins?

STM have an app note [AN4899](https://www.st.com/content/ccc/resource/technical/document/application_note/group0/13/c0/f6/6c/29/3b/47/b3/DM00315319/files/DM00315319.pdf/jcr:content/translations/en.DM00315319.pdf) that advises connecting all unused pins to GND: "If the application is sensitive to ESD, prefer a connection to ground".

There is however a risk specific to klipper with this. Klipper users can enable a pull-up on one of these pins by accident, creating a short to ground. This will probably cause that pin to burn out but it could do damage to other parts of the MCU in the process. The alternative is to use a resistor in the ground path to prevent this damage. This adds about 5 extra resistors around the MCU, which is space id rather not use up.

So the mitigation I have is to ship a klipper "pins file" which gives friendly names to all of the pins. This should discourage myself or someone else from doing a pull-up on the wrong pin. It also helps that NOTHING on the board requires a pull-up. All pull-ups have been done for you in hardware.

Most of the other chips have something similar in the documentation about connecting unused pins to ground.

### Test Points
Most of the items on the board can be gotten to via the rather generous pads for the connectors. Here is a list of everythign I want to scope and how im planning to get to it:

* GND - Via Connector Pad
* 3.3V - Test Point
* 5V Digital - TestPoint
* 5V Analog - TestPoint +  a local GND test point
