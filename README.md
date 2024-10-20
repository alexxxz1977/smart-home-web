# A battery-powered control panel for home automation with nanoFramework and Esp32
A few years ago, I became interested in home automation, specifically experimenting with Z-Wave devices for my home. Initially, I tried to integrate everything using existing solutions (for example, I tried Home Assistant & Domoticz), but then I decided to write my own application server. As a result, I wrote my own solution in 2019 using .Net Core 3.1, which I ran on a Raspberry Pi 3. Over time, I integrated additional elements into my solution, specifically adding the ability to control the server using SMS and adding integration with weather forecasting and bus schedule services. As a result, my smart home automation system has acquired the following form:

![test2 drawio](https://github.com/user-attachments/assets/a50b1015-1e11-4148-bfce-301c666d43e0)

Recently, I've been working on creating a control panel for smart home automation that can communicate with an application server over the MQTT protocol to manage various Z-Wave devices. The developed panel is a small, battery-powered device based on the [Esp32 TinyPico](https://www.tinypico.com/). It communicates with a .Net Core application server to control Z-Wave devices and display information like weather forecasts or public transportation schedules.

I am relatively new to [nanoFramework](https://github.com/nanoframework), started learning it about 4 years ago, although I've been programming in C# since the early 2000s. While working with *nanoFramework*, I've take a long path from simple tasks like blinking LEDs to creating my custom runtime images for Esp32 with interop assemblies (mostly by developing drivers for peripheral devices). I tried to write scalable code so that if necessary, other functionality could be easily added to the panel. At this point, the project is more or less complete and working, a brief demonstration is below.

https://github.com/user-attachments/assets/a22187cc-90f7-48bc-870d-df81fdb5c137

The panel is designed as a "thin client," meaning it doesn't directly control devices or connect to third-party services on the internet. Instead, it sends requests to the application server via MQTT and processes the responses. This allows me with minimal effort to have a view of any device that attached to Raspberry PI, such as Z-Wave devices through Z-Wave stick, an SMS modem, and various online services like weather forecasts.

The panel has a 400x240 pixel [Sharp MIP display](https://www.sharpsde.com/products/displays/model/ls027b7dh01/) and a 4x4 matrix keyboard. Most of the time the device sleeps, waking up either when you press a key or at scheduled intervals. Wake-up on a schedule is needed to periodically update the weather forecast on the screen or the clock in status bar. Instead of updating the weather forecast, you can configure it to periodically update the public transport schedule.

The majority of the panel's code is written in C#. However, the code that directly works with the hardware or peripheral devices is written in C and compiled separately as part of the runtime. The C# code communicates with the low-level code through [interop assemblies](https://jsimoesblog.wordpress.com/2018/06/19/interop-in-net-nanoframework/). This is a common approach in nanoFramework development when you need to access hardware directly or write high-performance code. 

Here are some of the interop assemblies I created:

* Softprise.Devices.Mcp23008 - a driver for the MCP23008 chip, which handles input from the keyboard and wakes up the microcontroller when a key is pressed.
* Softprise.Devices.Rv3028c7 - a driver for the RV-3028-C7 chip, which is real-time clock (RTC) and also generates a 1Hz signal needed for the display.
* Softprise.Esp32.Mqtt - an MQTT client which is built on top of MQTT library  in Esp32 IDF.
* Softprise.Esp32.NativeUtils - this provides a way to access a low-level features in the Esp32's native API (IDF) that are not available in nanoFramework.
* Softprise.Text.Jsmn – tiny and fast JSON parser. I created it because the built-in nanoFramework JSON parser wasn't able to handle large files when I started to work on this project.
* Softprise.NanoCRT - a graphics library, a slightly reworked code of AdafruitGFX, that I initially used for another project and later added support for the Sharp MIP display. As the situation with the JSON-parser, I had to create it because nanoFramework didn't have good graphics support at the time.
* Softprise.Devices.Max1704x – a driver for Max17048 chip, which manages the battery charge level.

The entire presentation layer of the nanoFramework application is written in C#, which calls primitives that are provided by NanoCRT for rendering. The architecture follows a simple design pattern called MVC (Model-View-Controller). Everything related to the non-visual part is designed as services that run in separate threads. The application has a service that handles keyboard events, a service that manages Wi-Fi connection, a service that is responsible for MQTT connection, and a system service that is responsible for the rest of the background processes. These services run independently but can communicate with each other through semaphores.

As the project grew, it became difficult to test the code by constantly deploying it to the physical device. To simplify this, I built an emulator that runs the application on Windows by rendering the presentation layer on a canvas of a WPF application.

![Screenshot 2024-10-19 234728](https://github.com/user-attachments/assets/a6c61bd2-8a29-4816-9920-005a856503cc)

This allows me to test the UI and simulate the keyboard without having to deploy the application on the actual device. The emulator also simulates the Wi-Fi and MQTT connections, allowing me to develop most of the application without access to the physical device. I only test the final code on the actual device once I'm confident it works.

As I mentioned the project has been in development for almost four years. Because of that, I haven't been able to keep it up with the new releases of nanoFramework. Some code (especially the C code) that worked fine on older versions has become unreliable after updating to newer versions. So, I decided to stick with runtime v100.5.0.14 (0xC5322585) and ESP IDF v4.1. It's possible that some libraries now are not relevant, and newer versions of nanoFramework have better libraries available or allows to do some things with the less pain.

I'm currently thinking about the future of this project. While working on it, I not only increased my skills in nanoFramework and Esp 32, but also have some progress in making electronics and hardware design. Since I have a lack of ideas what could be added to software part, I'm thinking about improving the hardware side of things. If the time allows me, I plan to experiment with EasyEDA and try my skills at designing printed circuit boards. This will let me create a full printed board that integrates all the components I use. If you're interested in seeing the hardware design, you can find the schematics and a history of making the hardware here. For now, I will take a time off to enjoy the results of my work but will continue fixing minor bugs until I have time for larger improvements. 

If you have any ideas for software features or questions about how the panel works, feel free to reach me out! I'm always open to feedback.

