# The Vision behind the Home Panel Builder project

In this article I will try to describe the vision behind the Home Panel Builder project. Lets start with a few definitions.

## Basics

### Home Panel

A home panel in the sense of the Home Panel Builder project is a device with a display and a touchscreen that you can use to display information from your home automation, but that you can also use to control your home or whatever you have connected.

There are various ready made home panels called touchpanel, smart panel, smart home panel or various other names, that provide more or less the same function, but are either relatively expensive or lock you into the ecosystem of the vendor, selling the home panel.

On the other hand there are a couple of inexpensive devices that come with display, touchscreen, a programmable ESP32 microcontroller, a few other peripherals and a housing, that makes the device look nice, that can be converted into a home panel. For an ambitionous hobbyist it is not a major problem, especially with a little help from AI, to create a simple firmware that can be loaded onto such a device. However it takes quite a bit of knowledge or confidence into the AI to do so. Working with something like ESPHome to create a firmware, though it is getting simpler with every major revision, is still not easy.

### Home Panel Builder

The idea behind the Home Panel Builder is to give a user a simple UI through which he can design his own dashboard or control panel. When the user is satisfied with his design, he can create a corresponding firmware through the push of a button, which is then loaded onto his device.

### The Origin Story

The original idea for the Home Panel was much simpler. What was needed was a wall clock for the bedroom. It needed to be legible during the night, without being intrusive, so it wouldn't keep you up at night. An E-paper display wouldn't do.

With home automation already in place it would also be a good idea to display the weather forecast in the morning, so that you don't need to consult with your mobile phone when trying to pick the right dress for the day. The resolution of one of those matrix LED panels is too low for that and using multiple draws to much current. Wouldn't want a noisy fan in the bedroom.

How about also using the panel to control your home? Change room temperature, open the shutters, turn on the underfloor heating in the bathroom. A touchscreen would be great.

So a device with a higher resolution with a touchscreen would have to be chosen. If that then looked cool, wouldn't that also be something for the living room? Then why not display some nice pictures during the day, if the panel is not used.

### Inspiration

The Home Panel Builder is in part inspired by [EspControl](https://jtenniswood.github.io/espcontrol/) by James Tenniswood. He has done amazing stuff with the device category, that I think is just right for something like the Home Panel. However the EspControl software follows a slightly different philosophy by trying to fit the entire design process for the panel into the firmware as well. The Home Panel Builder follows a slightly different path.

### ESPHome

In order not to have to create an entire ecosystem for the creation of the firmware for the Home Panel, the Home Panel Builder will try to make use of ESPHome as much as possible.

In the early stages this may mean that the Home Panel Builder creates a chunk of YAML that the user can then paste into ESPHome. At a later stage we may find a way to directly provide the YAML file to ESPHome, without direct user interaction. The gold standard would be for a full integration with ESPHome, where Home Panel Builder creates a YAML file, provides it to ESPHome and tells ESPHome to build and upload the firmware for the Home Panel automatically.

### Home Assistant

There are three integration points for the Home Panel Builder with Home Assistant.

1. The UI of the Home Panel Builder should be integrated into Home Assistant in the same way, that the ESPHome Builder is.
2. The Home Panel should have access to Home Assistant in order to get the information for the entities that it is supposed to display and to control functions in Home Assistant.
3. The Home Panel Builder should have access to Home Assistant in order to find entities and actionable actions withoud having to look up entity ids and paste or type them in Home Panel Builder.

## Technology

### Devices

The Home Panel Builder is supposed to create the firmware for devices with a display and touchscreen in ranges between four and ten inches.

The first target for the Home Panel Builder will be the Guition JC8012P4A1. This is an ESP32-P4 powered device with a 10.1 inch display and touchscreen.

### Firmware

The firmware is created by providing an appropriate YAML file to ESPHome and have it compiled by ESPHome and uploaded to the device.

### Home Panel Builder

The Home Panel Builder itself is a web application.

Home Panel Builder is written in C# running in a dotnet Docker image that can be executed as a container in Home Assistant as a Home Assistant App.

## Design

