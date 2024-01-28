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

Exit the project and restart the VSCode. Then we can see the platformIO tasks in our project:
![vscode after restarting](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ze2ggzuvlqo5k03rwer6.png)

Initialization is complete. Now we can start writing our code.

### note:

When I tried to use interface of the platformIO to create a new project, It didn't work properly. So I used the command line to create the project. I didn't understand why it didn't work. If you know the reason, please let me know in the comments.

## Setting Up Your First 'Thing' in AWS IoT Core with SSL Certificates

In this part, we're going to set up a 'thing' in AWS IoT Core. This 'thing' represents our ESP32 microcontroller in the AWS cloud. We'll also download the SSL certificates required for secure communication. Here's how we do it:

#### 1. **Log in to AWS IoT Core**:

- First, log in to your AWS Management Console and navigate to IoT Core.

#### 2. **Create a Policy**:

- In the AWS IoT Core console, navigate to 'Secure', and then 'Policies'.
- Click on ‘Create a policy’.
- Name your policy, for example, `iot_policy`.
- Set the policy document with the following JSON structure:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["iot:*"],
      "Resource": "*"
    }
  ]
}
```

- This policy allows your 'thing' to perform all actions (`iot:*`) on all resources (`*`).

#### 3. **Create a 'Thing'**:

- Go to the 'Manage' section and click on 'Things'.
- Choose 'Create' to start creating a new 'thing'.
- Select 'Create a single thing'.
- Click on 'next'.
- Follow the guided steps to name your 'thing' and complete its creation. For this guide, we'll name our 'thing' `esp32_test1`.
- Click on 'next'.
- Select 'Auto-generate a new certificate (recommended)'.
- Select the policy you created in the previous step (iot_policy).
- Click on 'Create thing'.

#### 4. **Download SSL Certificates**:

- Once your thing is created, AWS IoT Core will prompt you to create certificates. Click on 'Create certificates'.
- Download the following three essential files for your thing:
  - A certificate file (`blablabla-certificate.pem.crt`).
  - A private key file (`blablabla-private.pem.key`).
  - The Amazon Root CA 1 (`AmazonRootCA1.pem`) file.
- Make sure to save these files securely, as they are vital for your ESP32 to communicate securely with AWS IoT Core.

The file names are too long and you can change the names. So do I. I changed the names as follows:

- `blalblablabla-certificate.pem.crt` to `certificate.pem.crt`
- `blalblablabla-private.pem.key` to `private.pem.key`
- `AmazonRootCA1.pem` to `aws_cert_ca.pem`

After changing the name of the files We have 3 files to use in our project:

- `certificate.pem.crt`
- `private.pem.key`
- `aws_cert_ca.pem`

## Configuring ESP32 Filesystem for AWS IoT Core Connection

In this part, we're going to configure the ESP32 filesystem to connect to AWS IoT Core. We'll use the LittleFS file system to store the SSL certificates and the AWS IoT Core endpoint. Here's how we do it:

#### 1. **Create a 'data' Folder**:

In the root directory of your project, create a folder called `data`. This folder will contain all the files we need to store in the ESP32 filesystem.

**Note**: The `data` folder must be in the root directory of your project. Otherwise, the ESP32 will not be able to find the files.

![data folder](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s73jg6j1qt2tk9tugdxq.png)

The mqtt_config.json file will contain the AWS IoT Core endpoint, port, clientId, and topic to publish data.

```json
{
  "port": 8883,
  "host": "YOUR_AWS_IOT_CORE_ENDPOINT",
  "clientId": "esp_test1",
  "publishTopic": "esp32/sensor/test"
}
```

To get host information we need to go to the AWS IoT Core console and click on the `Settings`tab. Then we can see the endpoint information.

While connecting to AWS IoT Core, port 8883 is used for secure connection.

clientId is the name of the thing we created in AWS IoT Core.

publishTopic is the topic name that we will publish data to.

The wifi_config.json file will contain ssid and password.

```json
{
  "ssid": "your_wifi_name",
  "password": "your_wifi_password"
}
```
