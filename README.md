# Introduction

In this blog post, we're going to connect an ESP32 microcontroller to AWS IoT Core. We'll use DynamoDB for data storage and trigger AWS Lambda functions, all programmed in Golang. This guide is perfect for anyone interested in building IoT projects using ESP32 and AWS. We'll walk through each step, making it easy to follow along, even if you're new to these technologies. Let’s get started!

## Technologies and Techniques Used

In this project, we utilize a range of technologies and techniques to achieve seamless integration between the ESP32 microcontroller and AWS services. Here's a breakdown of what we use:

### Hardware Side:

- **PlatformIO**: An open-source ecosystem for IoT development.
- **Arduino**: The programming platform for our ESP32 microcontroller.
- **C++ (CPP)**: The programming language used for ESP32 development.
- **knolleary/pubsubclient**: A client library for MQTT messaging.
- **LittleFS**: A file system for handling files on the ESP32.

### Cloud and Backend Side:

- **AWS IoT Core**: Manages and connects IoT devices.
- **DynamoDB**: AWS's NoSQL database service for handling device data.
- **Lambda Function**: AWS service for running backend code in response to events.
- **Golang**: The programming language used for writing AWS Lambda functions.

## Creating a PlatformIO Project: Your First Step

First, let's create a folder for our project. We'll use PlatformIO to create a new project inside that folder. We'll call our project `esp32_to_aws_hardware_example`. Then we'll open the project in VSCode.

```bash
mkdir esp32_to_aws_hardware_example
cd esp32_to_aws_hardware_example
code .
```

Press `F1` to open the command palette and type `PlatformIO: New Terminal`. This will open a new terminal in VSCode. We'll use this terminal to run all the commands in this guide.

![PlatformIO: New Terminal](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g56cs0x16qy1tbwnoxzz.png)

So We can create a new project using the following commands:

First we need to delete all cache files. Then we can create a new project. This command is optional you can skip it.

```bash
pio system prune -f
```

Project initialization:

```bash
platformio init
```

After initialization, we can see the following files and folders in our project:
![after initialization](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/az8kz31xgkcli4gy2980.png)

## Adding Libraries and Board Informations to PlatformIO

In platformio.ini file we can add libraries and board informations. We need to add the following libraries to our project:

```ini
[env:esp32doit-devkit-v1]
platform = espressif32
board = esp32doit-devkit-v1
framework = arduino
monitor_speed=115200
upload_port=/dev/cu.usbserial-0001
lib_extra_dirs = lib
board_build.filesystem = littlefs
lib_deps =
    bblanchon/ArduinoJson@^6.18.5
    knolleary/pubsubclient@^2.8
```

At this point, I want to talk about the `lib_extra_dirs` and `board_build.filesystem` parameters. We need to add the `lib_extra_dirs` parameter to the platformio.ini file to use the libraries in the lib folder. We need to add the `board_build.filesystem` parameter to the platformio.ini file to use the LittleFS file system.

## Installing Packages

To install the libraries, we need to run the following command:

```bash
pio pkg install
```

Initialization is complete. Now we can start writing our code.

### note:

When I tried to use interface of the platformIO to create a new project, It didn't work properly. So I used the command line to create the project. I didn't understand why it didn't work. If you know the reason, please let me know in the comments.
