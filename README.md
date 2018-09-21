# Amazon FreeRTOS Workshop for the Texas Instruments CC3220SF

### Lab 1: Setup the Environment

In this lab, you will download the tools needed for the subsequent
sections. You can use the provided USB and let the download run in the
background. You will then register your TI CC3220SF-LAUNCHXL with AWS
IoT Core.

Registering your device involves:

- Creating a Policy
- Creating a Thing
- Generating the X.509 certificates for your CC3220SF-LAUNCHXL to
  communicate with AWS IoT Core services
- Attaching the Policy to the certificates

[![Lab1](images/lab1.png)](./Lab1.md "Lab 1")

### Lab 2: Run the Amazon FreeRTOS Sample Code

In this lab, you will download the Amazon FreeRTOS sample code from AWS IoT, import the sample code into TI Code Composer that you downloaded from section 1, configure the Amazon FreeRTOS sample with your WiFi settings and the certificates from Lab 1, and flash to the CC3220SF-LAUNCHXL via the Debug flow.

[![Lab2](images/lab2.png)](./Lab2.md "Lab 2")

### Lab 3: Using AWS IoT Core

In this lab, you will program Amazon FreeRTOS to use the sensors on the CC3220SF-LAUNCHXL board. First, you will be able to get the temperature sensor readings from the board and send to AWS IoT Core. Next, you will get the accelerometer sensor readings and send to AWS IoT Core. You will then learn how to use Rules Engine to send data to AWS IoT Analytics and view the data on Amazon QuickSight.

[![Lab3](images/lab3.png)](./Lab3.md "Lab 3")

### Lab 4: Using Device Shadows

In this lab, you will program Amazon FreeRTOS to change the on-board LED state based on accelerometer readings reaching a specific threshold. The standard way to perform this actuation through AWS IoT Core is by using Device Shadows. You will learn the details about Reported and Desired state and understand how to control access to the I2C bus between two tasks communicating with different I2C slaves.

[![Lab4](images/lab4.png)](./Lab4.md "Lab 4")

### Lab 5: Over the Air Updates

In this lab, you will program Amazon FreeRTOS to leverage Over the Air Updates (OTA) with the CC3220SF-LAUNCHXL board. The successful change in version will be visualized by updating the version reported state to the Device Shadow.

[![Lab5](images/lab5.png)](./Lab5.md "Lab 5")
