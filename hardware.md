Back in early 2020, I was browsing AliExpress when I spotted the ESP32-WROOM32-DEVKIT. I put it in my cart without much thought, thinking it would be a fun little project. Little did I know, it would ignite a passion for programming microcontrollers and electronics. 

I had some experience with the BrainPad from GHI Electronics, even creating a simple MQTT client using .Net MicroFramework to publish data from sensors on the board. But support for Microframework faded around 2015, halting my progress.

When I first started experimenting with the ESP32, I tried MicroPython but didn't get very far of blinking LEDs. However, I soon discovered .Net nanoFramework – a completely revamped version of .Net MicroFramework with excellent support for the ESP32 also having a great library collection, and a wealth of examples. As a .Net developer since the early 2000s, I was intrigued by the prospect of writing object-oriented code on a board with limited resources.

I began experimenting with different displays and sensors, porting Arduino libraries written in C to C#. This process helped me gain a deeper understanding of the framework. One of my first projects was a weather tracker that displayed forecasts from the internet on a 2.13-inch ePaper screen mounted on ESP32-TTGO T5 board. It had a simple functionality: it woke up every hour, fetched the latest forecast, and then returned to sleep.

During development, I realized that despite the board's claims of being battery-powered, it consumed a significant amount of energy, even in deep sleep mode. This meant a 1000 mA battery could only last for about a week to a week and a half. I wanted to create a device with a display and extended battery life that could operate in deep sleep mode.

I explored various boards advertised as low-power, eventually settling on the TinyPico Esp32 V2 from Unexpected Maker. This board not only had a small size but also boasted a reported sleep current of around 12 uA, making it ideal for a low-power project.

Another challenge I faced during designing my device was finding a low-power display. My vision was for the device to display a static image (like a weather forecast) while in sleep mode, minimizing battery drain. An ePaper display seemed perfect, but its low refresh rate (about a second for a full redraw) presented a problem. I wanted to implement different functions via a main menu on the display, which meant an interface that updated once per second would feel sluggish. I needed a display that offered: a) low power consumption, b) adequate size and c) acceptable refresh rate.

After exploring a range of display options, I settled on the 2.7” Sharp MIP Display mounted on the Adafruit board. It was the ideal choice, even though it was monochrome. For the prototype, I decided on the following hardware configuration:

1. **TinyPICO Esp32 V2 Microcontroller**: A fantastic board designed for battery-powered applications with optimized power consumption that has:
*	Esp32 PICO D4 dual-core processor at 240 MHz
*	2.4 GHz WiFi, BLE 4.2 + USB/Serial UART for programming
*	4MB SPI RAM - crucial for developing complex applications using nanoFramework
*	14 free pins for peripherals
*	Compact size of just 18mm x 32mm
2.	**Sharp MIP display 2.7" (LS027B7DH01A)**: Connected via SPI interface, with a resolution of 400x240 and an incredibly low static image power consumption of just 0.05 mW.
3.	**4x4 matrix keyboard**: Connected via the I2C interface of the MCP23008 chip.

The first prototype of my device was assembled with wires on a breadboard. Initially, I wrote the firmware for the device in C#, including the display driver. This allowed for rapid development, but the application's performance suffered. The main menu lagged when I pressed keys and felt sluggish overall.

To address this, I decided to rewrite the display driver in C, along with the graphics library I was using. While working on the weather tracker, I had ported the AdafruitGfx primitives library to C#. I attempted to reuse it for the Sharp MIP display, but the speed in C# was only sufficient for ePaper displays. This meant returning to the original C code and writing a new driver in C for the LS027B7DH01A display. I also rewrote the driver for the MCP23008 in C for improved performance.

From that point forward, I followed a consistent pattern for developing peripheral drivers in nanoFramework. A speed-critical low-level layer is written in C, which handled direct communication with the hardware. On top of this, there is a C# wrapper for easy access from the native application. This approach is known as interop assemblies.

Despite the complexity of writing interop assemblies it proved to be efficient. After writing a few examples, I could quickly port Arduino libraries written in C to nanoFramework with minor modifications. The primary challenge was debugging. I opted for the simpler method of adding debug messages to my code, rather than setting up a full debugging environment for nanoFramework. Initially I am using Arduino sketch to debug the C driver before making it as a part of an interop assembly.

_As a side note about interop assemblies, every change to the C code requires rebuilding the entire nanoFramework image, as the C code is compiled directly into the runtime. This means you need to rebuild the image if you want to include or exclude an interop assembly._

_Setting up the build environment is relatively straightforward, thanks to pre-built Docker containers and the ability to edit build configurations in Visual Studio Code. Once you've built the image, you can upload it using esptool. This method allows for uploading both the runtime and the native C# application simultaneously. It's particularly convenient if you're only modifying and testing the runtime itself, not the native C# code._

_Based on my experience with nanoFramework, delving into the internals of the runtime, particularly through studying interops, is beneficial for a few reasons:_

_*	**Performance**: Interops allow you to write speed-critical code, improving application performance._
_*	**Architectural** Understanding: By working with interops, you gain a deeper understanding of nanoFramework's architecture. This helps you troubleshoot issues effectively, particularly when dealing with hardware-related problems._
_*	**Code Samples**: The nanoFramework kernel serves as a valuable repository of code samples, which can be a great resource for learning and development._

_In summary, while the learning curve for nanoFramework is steep, exploring the kernel and interops is highly recommended for anyone looking to develop serious applications with nanoFramework. It provides a deeper understanding of the system and allows you to overcome performance limitations and debug effectively._

During development, I encountered an unexpected feature of the Sharp MIP display. In addition to requiring 5V power (which the Adafruit board already provided through a DC-DC converter), the display also needed signal inversion synchronization. This could be achieved programmatically by inverting bits at the start of each command block sent to the display, or physically by sending a 1 Hz signal to a specific pin on the display. Without this, the pixel voltage would gradually drain, causing the image to fade away.

This wasn't an issue while the microcontroller was active, as I could either continuously update the screen or send the signal through a controller pin. However, when the microcontroller was in sleep mode, I needed a different solution. Initially, I attempted to program the ESP32's ULP co-processor, but this approach proved to be too power-intensive, even in sleep mode. I then began searching for alternative boards with Sharp MIP display support and stumbled upon the Newt board.

Newt is a low-power, open-source, 2.7-inch IoT display powered by an ESP32-S2 and featuring SHARP's MiP screen technology. It seemed like the perfect solution to the device I had been trying to create from scratch. Although Newt was still in pre-order at the time, with shipments scheduled for mid-2022, I placed an order and started studying its hardware design.

I noticed that the screen refresh issue could be solved using the RTC RV-3028-C7 chip. This RTC chip is similar to the DS1307 in functionality, providing a clock, calendar, and timer. However, it also allows you to program it to generate a signal at a desired frequency through one of its pins. The chip itself has very low power consumption, making it ideal for a low-power device. I decided to order a development board with this RTC to start writing drivers while I waited for the Newt board to arrive.

In late August 2022, my Newt board finally arrived! The module impressed me with its features: low power consumption, a built-in speaker, integrated screen, RTC, 10 touch pads, a LiPo charger, and a low-battery monitor. It seemed like everything I had been trying to achieve with my prototype was already included!

However, after deploying my application, I discovered two unexpected challenges:

1. **Single-Core Performance**: The C# application ran noticeably slower on the single-core Esp32-S2 compared to the dual-core Esp32. In the dual-core Esp32, the runtime (C code) runs on one core, while the C# interpreter and native application run on the second. I hadn't anticipated this to be a problem when everything ran on the same core, as the C code only occupies 1-2% of the core, and most of the time, the first core is in sleep mode. But in reality, the performance difference was quite noticeable.
2. **Touch Button Lag**: Due to the single-core performance issue, the touch buttons (connected to TOUCH_1-TOUCH_10 pins on the Esp32) lagged significantly compared to the physical buttons in the matrix keyboard. This led to an intermittent UI menu, which was frustrating. 

After spending time working with Newt, I realized that I couldn't significantly improve the performance of my application. Therefore, I decided to continue developing my prototype. However, I wasn't disappointed with Newt; it's still a great device. If you're planning to use Arduino for coding, you can create quite impressive applications with Newt. Check out the examples already available for Newt for inspiration.

By the end of 2022, I had an MVP version of my prototype, albeit still implemented on a breadboard with a multitude of wires. It was time to move it to a PCB. Since I lacked experience in PCB design, I opted for a ready-made breadcrumb PCB and painstakingly soldered all the nodes together. 

I had to abandon the Adafruit board with its pre-mounted display because it would have made the device too bulky. I had already found a suitable case and wanted to fit within its dimensions. Luckily, the Adafruit board's schematic was publicly available, so I could implement my own solution. I simply soldered a 10-pitch 0.5 mm to 2.5 mm adapter to the PCB to connect the display ribbon cable. Then, I soldered components to the adapter outputs, following the schematic in the display manual. I also had to add a 3.3V to 5V DC-DC converter to supply the required voltage to the display, using the MAX619CPA+. It was impossible to use the 5V line from the ESP32 TinyPICO, as the microcontroller has separate 3.3V and 5V lines, and if it's powered by a battery, the 5V line is disabled. 

After assembling the first version of the board, including a 5V booster, a display connector, a mounted board with the RV-3028-C7, and an output for the matrix keyboard, I tested the device. The first display attached to board was a clone purchased from AliExpress, and as expected, its quality was subpar. For some reason, it only allowed a maximum SPI frequency of 9 MHz, which resulted in lagging UI. This display died after a couple of months, likely due to my own experimentation.

So, I ordered an original Sharp display, which allowed me to use an SPI frequency of 10.5 MHz, compared to the Adafruit board that supported up to 12 MHz. The issue here might have been the quality of my soldering and the use of numerous wires on the board.

With the board finally assembled and tested, it was time to tackle the case. I chose a translucent blue Hammond 156U case and cut out a hole for the display. The ready-made board with the 4x4 matrix keyboard didn't fit, so I had to build my own. 

There are plenty of schematics for matrix keyboards based on MCP23008 online, but finding the right buttons was a challenge. I tried numerous buttons from AliExpress, but cheap Chinese tactile buttons were often stiff to press and clicked loudly. After a dozen options, I finally settled on Japanese Alpine xxx buttons. They were round, small, and had a compact size, making them the perfect fit for my case. After cutting out holes for the buttons and mounting the keyboard, I had an almost fully assembled top section of the device. 

I managed to fit an Adafruit LiPo Fuel Gauge based on the MAX170048 into the remaining space. This chip monitors the battery discharge level and reports it through the I2C interface. On the other side of the case, I placed a 3000 mAh battery and a LiPo charger based on the BQ22222. It allows you to charge the battery via USB-C and can deliver up to 2A (configurable from 0.5 mA to 2A). Additionally, this charger doesn't get hot like the popular TP4056. The maximum temperature during charging was around 30°C. 

The result was a nicely compact and functional device that met all the requirements.
![Circuit (2)](https://github.com/user-attachments/assets/3005da13-718f-4e3b-b669-ca5d235dc215)

Here's how it looks inside:
![20241016_011936](https://github.com/user-attachments/assets/614aabdf-63bb-4b48-90f0-3e14b7840490)
![20241016_011513](https://github.com/user-attachments/assets/d2229dd5-577d-4beb-be87-77154ebfb3c0)

I'm not really a hardware engineer, so the assembly might not be perfect, but it works! 

The battery life is pretty good. If the device checks for updates every minute (like for the time), it lasts about a week to a week and a half. But if you only check every hour (like for weather updates), it can last for a couple of months.

The device uses a lot of power when the Wi-Fi is on (about 180mAh), but it uses very little when it's sleeping (about 0.5mAh). 

I think we could make it even more efficient by putting all the chips directly on the board and getting rid of all the wires. If I have time, I'll try to design a new board and see what I can do.

Let me know if you have any questions! I'm happy to explain anything in more detail.
