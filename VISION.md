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

The firmware is created by providing an appropriate device configuration YAML file to ESPHome. ESPHome can then compile it and upload the firmware to the device.

By creating a device configuration file for ESPHome, the Home Panel Builder can make use of all the supported devices from ESPHome. The main cornerstones being the `display`, `touchscreen`, `lvgl` to render the panels content and `scripts` to define local or remote activities, that can be triggered through the Home Panel.

### Home Panel Builder

The Home Panel Builder itself is a web application.

Home Panel Builder is written in C# built into a dotnet Docker image and run as a Docker container. The intention is, that the container can be executed in Home Assistant as a Home Assistant App.

## Design

### Storage

The Home Panel Builder uses the configuration folder of ESPHome (short: config folder) to share YAML files and to store assets and settings. For example:

* In a ESPHome Builder instance in Home Assistant that is the folder `/homeasssistant/esphome` on the host.
* In an ESPHome Device Builder desktop instance on Windows that is the folder `%USERPROFILE%\esphome`.
* In a clone of the `https://github.com/esphome/esphome` repository that is the `config` folder that you would typically create in your local copy to store your configuration files.

Home Panel Builder will store panel designs and its own settings in a subfolder called `.home-panel`.

Design files are called `{panel-name}.design.yaml`. Home Panel Builder will generate a `{panel-name}.yaml` file for the ESPHome Device Builder in the config folder.

### Home Panel UI

The UI of a Home Panel is based on `lvgl.pages`. There is at least one page, which is the Home Page. Other pages can be added and can be navigated to either by user interaction or when certain events occur, like the Home Panel becoming idle after a certain period of time, displaying a different page during certain hours, or turning the screen off, when everyone leaves the house.

Every page has the same basic layout. It consists of a narrow bar at the top, which is used to display a title or certain status information, called the title bar. The area below the title bar is the grid. The grid consists of rows and columns, where each intersection of a row with a column is a cell. The grid holds tiles, where each tile is rectangular with its width and height being multipes of the width and height of a cell. The size of the tile can therefore be specified in the number of columns and rows that it covers. E.g. a tile that has the width of three cells and the height of two cells may be referred to as a 3x2. Tiles must not overlap.

The number of rows and columns in the grid should be chosen so that a 1x1 tile roughly has the size of a key on a keyboard.

> [!NOTE]
> The screen of the Guition JC8012P4A1 is 1280 pixels wide and 800 pixels high in landscape orientation (default is portrait). The active display area is 216.58 mm x 135.36 mm. For a cell size of approximately 18 mm by 18 mm and a title bar height of 9 mm, that is 12 cells across times 7 cells down.

Tiles of size 1x1 (small tiles) should only be used to represent keys, e.g. on a security keypad or to display simple status information that can for example be displayed with an icon and optional color.

Regular tiles start at a size of 2x2. They can display an icon in the top left corner, a large state value in the top right and a small tile title at the bottom.

Large tiles start at a size of 4x4. In addition to the typical content of a regular tile, they can also display more elaborate content, like a weather forecast.

Each tile can have up to two associated actions. One is executed, when the tile is pressed for a short period of time. The other is executed, when the tile is pressed for a longer period of time.

Typical actions that can be associated with a tile press or long press:

* Navigate to a specific page
* Navigate to the previous page
* Navigate to the home page
* Perform a Home Assistant action (e.g. to turn on a light)
* Execute a pre-defined or custom script