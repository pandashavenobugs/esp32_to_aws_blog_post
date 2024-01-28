# Introduction

In this blog post, We're going to connect an ESP32 microcontroller to AWS IoT Core. We'll use DynamoDB for data storage and trigger AWS Lambda functions, all programmed in Golang. This guide is perfect for anyone interested in building IoT projects using ESP32 and AWS. We'll walk through each step, making it easy to follow along, even if you're new to these technologies. Let’s get started!

## Technologies and Techniques Used

In this project, We utilize a range of technologies and techniques to achieve seamless integration between the ESP32 microcontroller and AWS services. Here's a breakdown of what We use:

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

First, let's create a folder for our project. We'll use PlatformIO to create a new project inside that folder. We'll call our project `esp32_to_aws_hardware_example`. Then We'll open the project in VSCode.

```bash
mkdir esp32_to_aws_hardware_example
cd esp32_to_aws_hardware_example
code .
```

Press `F1` to open the command palette and type `PlatformIO: New Terminal`. This will open a new terminal in VSCode. We'll use this terminal to run all the commands in this guide.

![PlatformIO: New Terminal](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g56cs0x16qy1tbwnoxzz.png)

So We can create a new project using the following commands:

First We need to delete all cache files. Then We can create a new project. This command is optional you can skip it.

```bash
pio system prune -f
```

Project initialization:

```bash
platformio init
```

After initialization, We can see the following files and folders in our project:
![after initialization](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/az8kz31xgkcli4gy2980.png)

## Adding Libraries and Board Informations to PlatformIO

In platformio.ini file We can add libraries and board informations. We need to add the following libraries to our project:

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

To install the libraries, We need to run the following command:

```bash
pio pkg install
```

Exit the project and restart the VSCode. Then We can see the platformIO tasks in our project:
![vscode after restarting](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ze2ggzuvlqo5k03rwer6.png)

Initialization is complete. Now We can start writing our code.

### note:

When I tried to use interface of the platformIO to create a new project, It didn't work properly. So I used the command line to create the project. I didn't understand why it didn't work. If you know the reason, please let me know in the comments.

## Setting Up Your First 'Thing' in AWS IoT Core with SSL Certificates

In this part, We're going to set up a 'thing' in AWS IoT Core. This 'thing' represents our ESP32 microcontroller in the AWS cloud. We'll also download the SSL certificates required for secure communication. Here's how We do it:

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
- Follow the guided steps to name your 'thing' and complete its creation. For this guide, We'll name our 'thing' `esp32_test1`.
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

In this part, We're going to configure the ESP32 filesystem to connect to AWS IoT Core. We'll use the LittleFS file system to store the SSL certificates and the AWS IoT Core endpoint. Here's how We do it:

#### 1. **Create a 'data' Folder**:

In the root directory of your project, create a folder called `data`. This folder will contain all the files We need to store in the ESP32 filesystem.

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

To get host information We need to go to the AWS IoT Core console and click on the `Settings`tab. Then We can see the endpoint information.

While connecting to AWS IoT Core, port 8883 is used for secure connection.

clientId is the name of the thing We created in AWS IoT Core.

publishTopic is the topic name that We will publish data to.

The wifi_config.json file will contain ssid and password.

```json
{
  "ssid": "your_wifi_name",
  "password": "your_wifi_password"
}
```

## Writing the ESP32 Code for AWS IoT Core Connectivity

In this part, We're going to write the ESP32 code for connecting to AWS IoT Core. We'll use the PubSubClient library to connect to AWS IoT Core using MQTT.

First We need to create the models for the wifi, mqtt and certificate configurations. We'll create the following models in the model folder in the lib folder.

- `WifiCredentialModel.h`

```cpp
// lib/model/WifiCredentialModel.h
// WifiCredentialModel class definition.

#ifndef WIFICREDENTIALMODEL_H
#define WIFICREDENTIALMODEL_H

#include <Arduino.h>

class WifiCredentialModel
{
public:
    String ssid;
    String password;
    WifiCredentialModel() : ssid(""), password(""){};
    WifiCredentialModel(String ssid, String password) : ssid(ssid), password(password){};

    bool isEmpty()
    {
        return ssid == "" || password == "";
    }
};

#endif
```

- `MqttCredentialModel.h`

```cpp
// lib/model/MqttCredentialModel.h
// MqttCredentialModel class definition.

#ifndef MQTTCREDENTIALMODEL_H
#define MQTTCREDENTIALMODEL_H

#include <Arduino.h>

class MqttCredentialModel
{
public:
    int port;
    String host;
    String clientId;
    String publishTopic;

    MqttCredentialModel() : port(0), host(""), clientId(""), publishTopic(""){};
    MqttCredentialModel(int port, String host, String clientId, String publishTopic) : port(port), host(host), clientId(clientId), publishTopic(publishTopic){};

    bool isEmpty()
    {
        return port == 0 || host == "" || clientId == "" || publishTopic == "";
    }
};

#endif

```

- `CertificateCredentialModel.h`

```cpp
// lib/model/CertificateCredentialModel.h
// CertificateCredentialModel definition.

#ifndef CERTIFICATECREDENTIALMODEL_H
#define CERTIFICATECREDENTIALMODEL_H

#include <Arduino.h>

class CertificateCredentialModel
{
public:
    String ca;
    String certificate;
    String privateKey;

    CertificateCredentialModel() : ca(""), certificate(""), privateKey(""){};
    CertificateCredentialModel(String ca, String certificate, String privateKey) : ca(ca), certificate(certificate), privateKey(privateKey){};

    bool isEmpty()
    {
        return ca == "" || certificate == "" || privateKey == "";
    }
};

#endif
```

Now We can create a service that will read the wifi, mqtt and certificate configurations from the files and return the models. We'll create the following service in the service folder in the lib folder.

- `ConfigService.h`

```cpp
// lib/service/ConfigService.h
// ConfigService class definition.

#ifndef CONFIGSERVICE_H
#define CONFIGSERVICE_H

#include <Arduino.h>
#include "../model/CertificateCredentialModel.h"
#include "../model/MqttCredentialModel.h"
#include "../model/WifiCredentialModel.h"
#include <FS.h>

class ConfigService
{
private:
    fs::FS &fileSystem;

public:
    ConfigService(fs::FS &fileSystem) : fileSystem(fileSystem){};

    WifiCredentialModel getWifiCredential();
    MqttCredentialModel getMqttCredential();
    CertificateCredentialModel getCertificateCredential();
};

#endif

```

- `ConfigService.cpp`

```cpp
// lib/service/ConfigService.cpp
// ConfigService class implementation
#include "ConfigService.h"
#include <ArduinoJson.h>
#include <FS.h>

String readFile(fs::FS &fs, const char *path)
{
    Serial.printf("Reading file: %s\r\n", path);

    File file = fs.open(path, FILE_READ);
    if (!file)
    {
        Serial.println("Failed to open file for reading");
        return "";
    }
    return file.readString();
}

CertificateCredentialModel ConfigService::getCertificateCredential()
{
    String ca = readFile(fileSystem, "/certs/aws_cert_ca.pem");
    String certificate = readFile(fileSystem, "/certs/certificate.pem.crt");
    String privateKey = readFile(fileSystem, "/certs/private.pem.key");
    return CertificateCredentialModel(ca, certificate, privateKey);
}

WifiCredentialModel ConfigService::getWifiCredential()
{
    File wifiConfigFile = fileSystem.open("/wifi_config.json", "r");
    if (!wifiConfigFile)
    {
        Serial.println("Failed to open wifi_config.json file");
        return WifiCredentialModel();
    }
    DynamicJsonDocument doc(1024);
    DeserializationError error = deserializeJson(doc, wifiConfigFile);
    if (error)
    {
        Serial.println("Failed to read file, using default configuration");
        wifiConfigFile.close();
        return WifiCredentialModel();
    }
    String ssid = doc["ssid"];
    String password = doc["password"];
    wifiConfigFile.close();
    return WifiCredentialModel(ssid, password);
}

MqttCredentialModel ConfigService::getMqttCredential()
{
    File mqttConfigFile = fileSystem.open("/mqtt_config.json", "r");
    if (!mqttConfigFile)
    {
        Serial.println("Failed to open mqtt_config.json file");
        return MqttCredentialModel();
    }
    DynamicJsonDocument doc(1024);
    DeserializationError error = deserializeJson(doc, mqttConfigFile);
    if (error)
    {
        Serial.println("Failed to read file, using default configuration");
        mqttConfigFile.close();
        return MqttCredentialModel();
    }
    String host = doc["host"];
    int port = doc["port"];
    String clientId = doc["clientId"];
    String publishTopic = doc["publishTopic"];
    mqttConfigFile.close();
    return MqttCredentialModel(port, host, clientId, publishTopic);
}
```

We read the files in the data folder. Then We deserialize the JSON data using the ArduinoJson library. We return the models We created using the data We read.

## Bringing It All Together in main.cpp: Finalizing the ESP32-AWS Connection

In this part, We're going to write the main.cpp file. We'll use the PubSubClient library to connect to AWS IoT Core using MQTT. We'll use the ConfigService We created to read the wifi, mqtt and certificate configurations from the files. We'll use the models We created to store the configurations.

- `main.cpp`

```cpp
#define LED_PIN 2
#include <Arduino.h>
#include <LittleFS.h>
#include <PubSubClient.h>
#include <WiFiClientSecure.h>
#include "../service/ConfigService.h"

WiFiClientSecure espClient;
PubSubClient client(espClient);
MqttCredentialModel mqttCredential;
WifiCredentialModel wifiCredential;
CertificateCredentialModel certificateCredential;

void setup()
{
    Serial.begin(115200);
    Serial.println("Starting...");
    if (!LittleFS.begin())
    {
        Serial.println("An Error has occurred while mounting LittleFS");
        return;
    }

    ConfigService configService(LittleFS);
    wifiCredential = configService.getWifiCredential();
    if (wifiCredential.isEmpty())
    {
        Serial.println("Wifi credential is empty");
        return;
    }

    mqttCredential = configService.getMqttCredential();
    if (mqttCredential.isEmpty())
    {
        Serial.println("Mqtt credential is empty");
        return;
    }

    certificateCredential = configService.getCertificateCredential();
    if (certificateCredential.isEmpty())
    {
        Serial.println("Certificate credential is empty");
        return;
    }

    WiFi.begin(wifiCredential.ssid.c_str(), wifiCredential.password.c_str());
    while (WiFi.status() != WL_CONNECTED)
    {
        digitalWrite(LED_PIN, HIGH);
        delay(50);
        digitalWrite(LED_PIN, LOW);
        delay(50);
        digitalWrite(LED_PIN, HIGH);
        delay(50);
        digitalWrite(LED_PIN, HIGH);
        delay(1000);
        Serial.print(".");
    }
    Serial.println("WiFi connected");

    // Set the certificates to the client
    espClient.setCACert(certificateCredential.ca.c_str());
    espClient.setCertificate(certificateCredential.certificate.c_str());
    espClient.setPrivateKey(certificateCredential.privateKey.c_str());

    client.setServer(mqttCredential.host.c_str(), mqttCredential.port);

    while (!client.connected())
    {
        Serial.println("Connecting to AWS IoT...");

        if (client.connect(mqttCredential.clientId.c_str()))
        {
            Serial.println("Connected to AWS IoT");
        }
        else
        {
            Serial.print("failed, rc=");
            Serial.print(client.state());
            Serial.println(" try again in 5 seconds");
            delay(5000);
        }
    }
}
void loop() {}
```

## Testing the ESP32-AWS Connection

To test esp32-aws connection, We need to build and upload the code and folders to the esp32. We can do this using the platformIO tasks. We need to run the following tasks

- Build Filesystem Image
- Upload Filesystem Image
- Build Code
- Upload Code
- Monitor

To build Filesystem Image We need to run Build Filesystem Image task:

![Build FileSystem Image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7jc1uuersr3xlolzp006.png)

After the Clicking the Build Filesystem Image task, We can see the following output in the terminal:

![Build FileSystem Image Output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6fz32aefmw3v1kgqz9z1.png)

Now You can upload the filesystem image to the esp32. To do this, We need to run the Upload Filesystem Image task:

The output of the Upload Filesystem Image task is as follows:

![Upload FileSystem Image Output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ibgh3pec3yk5j3foqgc9.png)

Now We can build the code. To do this, We need to run the Build task:

![Build Code](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ve1qvyvgee5hnt3ca2fm.png)

ESP32 is ready to upload the code. To do this, We need to run the Upload Code task but I go with the Upload and Monitor task to see the output after uploading the code.

After clicking the Upload and Monitor task, We can see the following output in the terminal:

![After Upload and Monitor](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/700bu8oqjeb5lsz0gvqn.png)

As you can see, the esp32 is connected to AWS IoT Core. Now We can send data to AWS IoT Core.

## Sending Data to AWS IoT Core

In this part, We're going to send data to AWS IoT Core. I have a dht11 sensor and I want to send the temperature and humidity data to AWS IoT Core. I'll use the following libraries to read the data from the sensor:

- `Adafruit Unified Sensor`
- `DHT sensor library`

We need to add the following libraries to our project:

```ini
lib_deps =
    bblanchon/ArduinoJson@^6.18.5
    knolleary/pubsubclient@^2.8
    adafruit/DHT sensor library@^1.4.6
    adafruit/Adafruit Unified Sensor@^1.1.14
```

After adding these libraries, platformIO will download the libraries and add them to the lib folder. If not, you need to open platformIO terminal and run the following command:

```bash
pio pkg install
```

First, We need to create a model for the sensor data. We'll create the payload model in the model folder in the lib folder.

- `PayloadModel.h`

```cpp
// lib/model/PayloadModel.h
// PayloadModel class definition.

#ifndef PAYLOADMODEL_H
#define PAYLOADMODEL_H

#include <Arduino.h>
#include <ArduinoJson.h>

class PayloadModel
{
private:
    String clientId;
    bool isClientIdValid;
    float humidity;
    bool isHumidityValid;
    float temperature;
    bool isTemperatureValid;

public:
    PayloadModel()
    {
        clientId = "";
        isClientIdValid = false;
        humidity = 0;
        isHumidityValid = false;
        temperature = 0;
        isTemperatureValid = false;
    };
    void setClientId(String clientId, bool isClientIdValid)
    {
        this->clientId = clientId;
        this->isClientIdValid = isClientIdValid;
    };
    void setHumidity(float humidity, bool isHumidityValid)
    {
        this->humidity = humidity;
        this->isHumidityValid = isHumidityValid;
    };
    void setTemperature(float temperature, bool isTemperatureValid)
    {
        this->temperature = temperature;
        this->isTemperatureValid = isTemperatureValid;
    };

    char *toJson()
    {
        static char buffer[512];
        DynamicJsonDocument doc(256);
        if (this->isClientIdValid)
        {
            doc["clientId"] = this->clientId;
        }
        else
        {
            doc["clientId"] = nullptr;
        }
        if (this->isHumidityValid)
        {
            doc["humidity"] = this->humidity;
        }
        else
        {
            doc["humidity"] = nullptr;
        }
        if (this->isTemperatureValid)
        {
            doc["temperature"] = this->temperature;
        }
        else
        {
            doc["temperature"] = nullptr;
        }

        serializeJson(doc, buffer);
        return buffer;
    };
};
#endif

```

For the payloadModel We need to create a json string. We'll use the ArduinoJson library to create the json string.

Now We can implement the payloadModel and dht11 sensor in the main.cpp file.

- `main.cpp`

```cpp
#define LED_PIN 2
#define DHT_PIN 15
#define DHT_TYPE DHT11

unsigned long previousSensorMillis = 0;
const long sensorInterval = 10000;

#include <Arduino.h>
#include <LittleFS.h>
#include <PubSubClient.h>
#include <WiFiClientSecure.h>
#include "../service/ConfigService.h"
#include "../model/PayloadModel.h"
#include <DHT.h>
WiFiClientSecure espClient;
PubSubClient client(espClient);
MqttCredentialModel mqttCredential;
WifiCredentialModel wifiCredential;
CertificateCredentialModel certificateCredential;
PayloadModel payloadModel;
char *payload;
DHT dht(DHT_PIN, DHT_TYPE);

void setup()
{
    Serial.begin(115200);
    Serial.println("Starting...");
    if (!LittleFS.begin())
    {
        Serial.println("An Error has occurred while mounting LittleFS");
        return;
    }

    ConfigService configService(LittleFS);
    wifiCredential = configService.getWifiCredential();
    if (wifiCredential.isEmpty())
    {
        Serial.println("Wifi credential is empty");
        return;
    }

    mqttCredential = configService.getMqttCredential();
    if (mqttCredential.isEmpty())
    {
        Serial.println("Mqtt credential is empty");
        return;
    }

    certificateCredential = configService.getCertificateCredential();
    if (certificateCredential.isEmpty())
    {
        Serial.println("Certificate credential is empty");
        return;
    }

    WiFi.begin(wifiCredential.ssid.c_str(), wifiCredential.password.c_str());
    while (WiFi.status() != WL_CONNECTED)
    {
        digitalWrite(LED_PIN, HIGH);
        delay(50);
        digitalWrite(LED_PIN, LOW);
        delay(50);
        digitalWrite(LED_PIN, HIGH);
        delay(50);
        digitalWrite(LED_PIN, HIGH);
        delay(1000);
        Serial.print(".");
    }
    Serial.println("WiFi connected");

    // Set the certificates to the client
    espClient.setCACert(certificateCredential.ca.c_str());
    espClient.setCertificate(certificateCredential.certificate.c_str());
    espClient.setPrivateKey(certificateCredential.privateKey.c_str());

    client.setServer(mqttCredential.host.c_str(), mqttCredential.port);

    while (!client.connected())
    {
        Serial.println("Connecting to AWS IoT...");

        if (client.connect(mqttCredential.clientId.c_str()))
        {
            Serial.println("Connected to AWS IoT");
        }
        else
        {
            Serial.print("failed, rc=");
            Serial.print(client.state());
            Serial.println(" try again in 5 seconds");
            delay(5000);
        }
    }
    // set the payload model
    payloadModel = PayloadModel();
    payloadModel.setClientId(mqttCredential.clientId, true);
}
void loop()
{
    unsigned long currentMillis = millis();
    if (currentMillis - previousSensorMillis >= sensorInterval)
    {
        digitalWrite(LED_PIN, HIGH);
        previousSensorMillis = currentMillis;
        float humidity = dht.readHumidity();
        float temperature = dht.readTemperature();
        payloadModel.setHumidity(humidity, !isnan(humidity));
        payloadModel.setTemperature(temperature, !isnan(temperature));
        payload = payloadModel.toJson();
        Serial.println("Publish message: ");
        Serial.println(payload);
        client.publish(mqttCredential.publishTopic.c_str(), payload);
    }
    digitalWrite(LED_PIN, LOW);
}
```

This code reads the temperature and humidity data from the sensor and sends it to AWS IoT Core every 10 seconds as json data.

To test the code run the Upload and Monitor task. After uploading the code, We can see the following output in the terminal:

![Sending data to aws iot core](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1059sgh02l5nls1zjfsr.png)

As We can see, the esp32 is sending data to AWS IoT Core.

We can check if the esp32 sends data to AWS IoT Core. To do this, We need to go to the AWS IoT Core console and click on the `Test` tab. Then We can subscribe to the topic We created in the mqtt_config.json file.

![Subscribe to the topic](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hpiv56boe63ph7r1siig.png)

After subscribing to the topic, We can see the data sent by the esp32.

![getting mqtt data](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4uw1zxlvh27xjafymfor.png)

The data we can see is as follows:

```json
{
  "clientId": "esp_test1",
  "humidity": 55,
  "temperature": 27.60000038
}
```

Everything is working properly. So What is next?
