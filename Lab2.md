# Amazon FreeRTOS Workshop for the Texas Instruments CC3220SF
## Lab 2: Run the Amazon FreeRTOS Sample Code

In this lab, you will download the Amazon FreeRTOS sample code from Github, import the sample code into TI Code Composer that you downloaded from section 1, configure the Amazon FreeRTOS sample with your WiFi settings and the certificates from section 2, and download to the CC3220SF-LAUNCHXL via the Debug flow.

### Initializing the Repository

In this section, you will learn the physical directory layout for Amazon FreeRTOS.  You will clone the git repository that include the Code Composer Studio projects and a reference to Amazon FreeRTOS as a git repository submodule.

From your Home directory, check out the sources.

1. Open a console window on your workstation to perform work on the command line.
2. Clone the repository.
```
git clone --recursive https://github.com/rpcme/afr-workshop-cc3220sf
```
3. In your HOME directory, you should now have two directories: ```aws-iot``` and ```afr-workshop-cc3220sf```.

From this point onward, and throughout the remaining labs, the repository checkout directory is referred to as ```LAB_REPOPATH```.

Alternatively, if you don't have git or you don't want to clone the repository (which is the best option, perform these steps:

1. Navigate to https://github.com/rpcme/afr-workshop-cc3220sf.
2. Download the repository as ZIP.  Extract.  This will be known as ```LAB_REPOPATH```.
3. Navigate to https://github.com/aws/amazon-freertos.
4. Download the repository as ZIP.  Extract to ```LAB_REPOPATH/amazon-freertos```.  ```LAB_REPOPATH/amazon-freertos``` should have ```demos```, ```lib```, ```tests```, and ```tools``` directories.

### Open and Build the Amazon FreeRTOS Sample Project

In this section, open the Amazon FreeRTOS Sample Project that serves as the baseline for the labs.

1. Open Code Composer Studio 8.0 (CCS8).  When you first open CCS8, the IDE will prompt your for a workspace location. Choose the default location and check the box "don't ask again".
2. Import the project.
   1. Open the import dialog by clicking **File > Import...**.
2. Change to the latest compiler version available.
3. Run a full compile by clicking Project > Build All.

At this point, the project builds but it is not configured properly for connectivity.  In the next section, we will configure the project to run the Hello World demo.

### Configuring for the Hello World Demo

In CCS8, open the file ```application_code``` > ```common_demos``` > ```include``` > ```aws_clientcredential.h```.

2. Edit the ```clientcredentialMQTT_BROKER_ENDPOINT``` variable on line 38.  To retrieve the value for this variable, issue the following command using the AWS CLI.

   ```bash
   aws iot describe-endpoint
   ```
   
   The output will be similar to the following.  **Use the output from the command on your workstation, not this value.  Endpoints can differ between AWS accounts.***
   
   ```text
   audqth7zumq6e.iot.us-east-1.amazonaws.com
   ```
   
   The result on line 38 will be similar to the following.
   
   ```c
   static const char clientcredentialMQTT_BROKER_ENDPOINT[] = "audqth7zumq6e.iot.us-east-1.amazonaws.com";
   ```
   
3. Edit the ```clientcredentialIOT_THING_NAME``` variable on line 43.  Although this value is used for the Device Shadow Client, we will edit it for completeness.  In the previous lab, you identified the MAC address for your device. For example, it may be similar to ```9884e3f60411```.  The value will then look similar to the following.

   ```c
   #define clientcredentialIOT_THING_NAME "9884e3f60411"
   ``` 
4. Edit the ```clientcredentialWIFI_SSID``` variable on line 58.  For example, if your SSID is ``MySuperWiFi", then the value will look similar to the following.

   ```c
   #define clientcredentialWIFI_SSID       "MySuperWiFi"
   ```
5. Edit the ```clientcredentialWIFI_PASSWORD``` variable on line 63.  For example, if your Wi-Fi password is ```p@s$woRd```, then the value will look similar to the following.

   ```c
   #define clientcredentialWIFI_PASSWORD   "p@s$woRd"
   ```
   
Next, we need to generate the header file that holds credentials for connecting to AWS IoT Core.  In CCS8, open the file ```application_code``` > ```common_demos``` > ```include``` > ```aws_clientcredential_keys.h```.

On your workstation, open Explorer (Windows) or Finder (Mac OSX) and navigate to ```$LAB_REPOPATH``` > amazon-freertos > demos > common > ```devmode_key_provisioning``` > ```CertificateConfigurationTool```.  Double-click ```CertificateConfigurator.html```.

1. For **Certificate PEM file**, click **Browse...** and navigate to ```$LAB_REPOPATH``` > ```credentials```.  Select the certificate file, which will look similar to ```9884e3f60411_certificate.pem```.
2. For **Private Key PEM file**, click **Browse...** and navigate to ```$LAB_REPOPATH``` > ```credentials```.  Select the private key file, which will look similar to ```9884e3f60411_private_key.pem```.
3. Click **Generate and save ```aws_clientcredential_keys.h```**.  Save the file to the filesystem.  I most cases, it will be saved to your Downloads directory.
4. Copy the ```aws_clientcredential_keys.h``` in your Downloads directory ***over*** the ```aws_clientcredential_keys.h``` in the directory ```$LAB_REPOPATH``` > amazon-freertos > demos > common > ```include```.
5. In CCS8, the file should already be refreshed.  If not, double-click on aws_clientcredential_keys.h again.  It should look similar to the following (the content has been modified so this is for example purposes only).

   ```c
	/*
	 * PEM-encoded client certificate.
	 *
	 * Must include the PEM header and footer:
	 * "-----BEGIN CERTIFICATE-----"
	 * "...base64 data..."
	 * "-----END CERTIFICATE-----";
	 */
	static const char clientcredentialCLIENT_CERTIFICATE_PEM[] = 
	"-----BEGIN CERTIFICATE-----\n"
	"MIIDWTCCAkGgAwIBAgIUBrxwa1bQrALUSLuVzFLgUNvLMuwwDQYJKoZIhvcNAQEL\n"
	"BQAwTTFLMEkGA1UECwxCQW1hem9uIFdlYiBTZXJ2aWNlcyBPPUFtYXpvbi5jb20g\n"
	"SW5jLiBMPVNlYXR0bGUgU1Q9V2FzaGluZ3RvbiBDPVVTMB4XDTE4MDkxOTE0NTIy\n"
	"v8oxIkX5xzR2iQ0RbM0IyroTp/WpTe5bLLwSOsWujeElC9PgLw+CB6WEZAlNxNrx\n"
	"fPSkKCqOF+WztJd2N8SFUlkr1uvzJrcy0L0V6HNdYFGDfUzal/IcV6cNadIgWu6m\n"
	"KSsrSqSGa+fxxyc/mCw83O5pm887dLVysnPB1EGTuGKLacP3QVJIrnJTasGb\n"
	"-----END CERTIFICATE-----";
	
	/*
	 * PEM-encoded client private key.
	 *
	 * Must include the PEM header and footer:
	 * "-----BEGIN RSA PRIVATE KEY-----"
	 * "...base64 data..."
	 * "-----END RSA PRIVATE KEY-----";
	 */
	static const char clientcredentialCLIENT_PRIVATE_KEY_PEM[] =
	"-----BEGIN RSA PRIVATE KEY-----\n"
	"MIIEowIBAAKCAQEAuRdLQNfQl+zKDu/FAQOGpHt8psX0b22QdS/6vQLFMhVniJhG\n"
	"7pzKC/IFAMYfW8r6kNrb5rWYZ73BCVW/V4Eu4nXtCoiZ6qiO7vtEg8uwvFOywBbV\n"
	"Ny6057Vf4dD7r5pVQkYat14fLkxm4lbB0Ot07gwisRNbTA69AxzkPk2YydZPM8fx\n"
	"icWLxbnkWwUM2qLMinjNZ+ifdmu9ZUCdHa4DeYs+yneK4HyhyA/y1iHE77mA6rv/\n"
	"EoCtyUVWXjoJKiR47TCApvuQPuN+MUnuclylg5Ipm0UPGhEh08iFlQL5wqJtLOKo\n"
	"jw8HTlzChZlxRR7sX8xscJpNOMMJMHn6w1b+SQIDAQABAoIBAGJlV8Qo9uNNJj26\n"
	"iWh4j2NxolqmccOq41jpIMrKZjqx1lBvY4YGJAMlrjMzIAiUZ5AKSUzkzRiZb72e\n"
	"sBG8w0fwmQs4g1qUMSgqu8m0x0fW1vf3K3IiilmDum05xBMeYKTSz/EbQEhEpY09\n"
	"Nyg7U2MvEsULcdz6wOLJ14OC5YCKo5LQvJ7W+7qXb7cAR0VuZbgnSj+iUQhunnsz\n"
	"GiLiLurrK+TI1bhZR1SGuQKBgDtkBzMGr8dnVSnVL5YkkcDUui5ZzUq8DDIXiRmp\n"
	"j0UPPmsuhCK8ZoxEcYt43BdwqaZbs1895rznLeks4evMnsxN6ZLYw8C4rXugZK5q\n"
	"RYBh/mimXorVp5cWTUaWgpEOSMTi7OVil9m5leDbouB8u6tSWuNiXt1MhVAU1fn+\n"
	"Iq1VAoGBAImYjzoWqsKowPF8xcqHg+C2ym2VeDEramB3VvXUDHf4jx+Ma/m/FMEM\n"
	"3qlix7Nks/N8+4XGXBod4fxRAhp9Eza/YjFlKhNqW1jzTi0p5g8iySbe6J7H1jPO\n"
	"th5obcRvBds7fPzbkiApuT7zhUpskXuQLTpsBpLfhHSMnXQzOtxT\n"
	"-----END RSA PRIVATE KEY-----";
	
	/*
	 * PEM-encoded Just-in-Time Registration (JITR) certificate (optional).
	 *
	 * If used, must include the PEM header and footer:
	 * "-----BEGIN CERTIFICATE-----"
	 * "...base64 data..."
	 * "-----END CERTIFICATE-----";
	 */
	static const char * clientcredentialJITR_DEVICE_CERTIFICATE_AUTHORITY_PEM = NULL;
   ```

### Testing the connection

1. Change the SOP jumper to SOP Position 0 (JTAG).
2. Reset the board.
3. Click the debug link.
4. If you get a dialog box to upgrade the XDS-110 Firmware, click Update.  After updating, unplug the board, then plug in the board, and then click debug again.
5. When the debug process starts, your debugging session will start and an automatic breakpoint will be added for the ```main()``` function.
6. Before continuing, login to the AWS Console, and then navigate to the AWS IoT Core console.
7. On the left-hand side, click the **Test** menu item.
8. For the subscription topic, enter ```#```
9. Click the ```Subscribe to topic``` button.
10. Open a Terminal window from CCS8.  First, we need to add the view.  Navigate to Window > Show View > 
10. Switch to CCS8.  Click the Resume button **TODO ADD IMAGE FOR BUTTON**
11. The demo should now be running.  When completed, the output to the CCS8 terminal will be similar to the following **TODO ADD IMAGE**

In the following sections, we will modify the sample program to do something a bit more interesting with it.

### Modifying the Client ID

When connecting to AWS IoT Core, the MQTT Client ID that you are using for the MQTT sessopm must be unique in your account.  If not unique, then the last connection made with the Client ID will cause a disconnect for the previous connection.

In these labs, we have established a unique ID - the MAC address.  If you look at the source file ```application_code``` > ```common_demos``` > ```source``` > ```aws_hello_world.c```, you will note that on line 85 the Client ID is hardcoded.  If we are going to use the same firmware image for hundreds, thousands, or even millions of devices, we cannot have this value hardcoded.

In the Wi-Fi abstraction layer in Amazon FreeRTOS, there is an API to retrieve the MAC Address.  We can use that to construct our unique identifier.

1. At the bottom of ```aws_hello_world.c```, paste the following code.  

   ```c
   static void prvMacForHumans(uint8_t * humanAddress)
	{
	    uint8_t buf[ wificonfigMAX_BSSID_LEN ];
	    char cbuf[2];
	    uint8_t idx = 0;
	
	    WIFI_GetMAC( ( uint8_t * )  buf);
	    
	    // get the mac address in a string
	    for (idx = 0; idx < wificonfigMAX_BSSID_LEN; idx++)
	    {
	        sprintf(cbuf, "%x", buf[idx]);
	        if ( cbuf[1] == '\0')
	        {
	            humanAddress[( idx * 2 )] = '0';
	            humanAddress[( idx * 2 ) + 1] = cbuf[0];
	
	        }
	        else
	        {
	            humanAddress[( idx * 2 )] = cbuf[0];
	            humanAddress[( idx * 2 ) + 1] = cbuf[1];
	        }
	    }
	
	    humanAddress[ ( wificonfigMAX_BSSID_LEN * 2 ) ] = '\0';
	}
   ```
2. On line 72, add the following include. 

   ```c
   #include "aws_wifi.h"
   ```
3. On line 182, add the following declarations.

   ```c
   static void prvMacForHumans(uint8_t * humanAddress);
   static uint8_t thing_mac_address[ ( wificonfigMAX_BSSID_LEN * 2 ) + 1 ];
   ```
4. Next, we need to initialize the value for ```thing_mac_address```.  We can do this in the entry function to this demo.  Scroll to line 512, and enter the line for assigning the value to ```thing_mac_address```.  Existing code has been added here for context.

   ```c
   configPRINTF( ( "Creating MQTT Echo Task...\r\n" ) );

   prvMacForHumans(thing_mac_address);

   /* Create the message buffer used to pass strings from the MQTT callback
   ```
5. Finally, we need to edit the Client ID for the MQTT client.  Scroll to line 187, to this function:

   ```c
   static BaseType_t prvCreateClientAndConnectToBroker( void )
   ```
6. Four lines down, a structure is created to hold the MQTT connection parameters.  The structure should look like the following:

   ```c
        MQTTAgentConnectParams_t xConnectParameters =
	    {
	        clientcredentialMQTT_BROKER_ENDPOINT, /* The URL of the MQTT broker to connect to. */
	        democonfigMQTT_AGENT_CONNECT_FLAGS,   /* Connection flags. */
	        pdFALSE,                              /* Deprecated. */
	        clientcredentialMQTT_BROKER_PORT,     /* Port number on which the MQTT broker is listening. Can be overridden by ALPN connection flag. */
	        echoCLIENT_ID,                        /* Client Identifier of the MQTT client. It should be unique per broker. */
	        0,                                    /* The length of the client Id, filled in later as not const. */
	        pdFALSE,                              /* Deprecated. */
	        NULL,                                 /* User data supplied to the callback. Can be NULL. */
	        NULL,                                 /* Callback used to report various events. Can be NULL. */
	        NULL,                                 /* Certificate used for secure connection. Can be NULL. */
	        0                                     /* Size of certificate used for secure connection. */
	    };
    ```
    To change the Client ID, we will change ```echoCLIENT_ID``` to ```thing_mac_address```.  It will look like the following:
    
    ```c
        MQTTAgentConnectParams_t xConnectParameters =
	    {
	        clientcredentialMQTT_BROKER_ENDPOINT, /* The URL of the MQTT broker to connect to. */
	        democonfigMQTT_AGENT_CONNECT_FLAGS,   /* Connection flags. */
	        pdFALSE,                              /* Deprecated. */
	        clientcredentialMQTT_BROKER_PORT,     /* Port number on which the MQTT broker is listening. Can be overridden by ALPN connection flag. */
	        thing_mac_address,                        /* Client Identifier of the MQTT client. It should be unique per broker. */
	        0,                                    /* The length of the client Id, filled in later as not const. */
	        pdFALSE,                              /* Deprecated. */
	        NULL,                                 /* User data supplied to the callback. Can be NULL. */
	        NULL,                                 /* Callback used to report various events. Can be NULL. */
	        NULL,                                 /* Certificate used for secure connection. Can be NULL. */
	        0                                     /* Size of certificate used for secure connection. */
	    };
   ```
7. Build and debug-run the application to ensure everything is working with your new unique Client ID.  How do we know that we are using the ID?  We can add logging instrumentation to help us know that.  Before the structure definition, add this line:

   ```c
   configPRINTF( ( "My Client ID is [%s]\r\n", thing_mac_address ) );
   ```

   in the Terminal logging window, you will see the following:
   
   ```
   8 1670 [Tmr Svc] Wi-Fi connected to AP Gwypo_2.4GHz_1878.
9 1670 [Tmr Svc] IP Address acquired 192.168.29.199
10 1671 [Tmr Svc] Creating MQTT Echo Task...
11 1672 [MQTTEcho] My Client ID is [9884e3f60411]
12 1672 [MQTTEcho] MQTT echo attempting to connect to audqth7zumq6e.iot.us-east-1.amazonaws.com.
   ```
Next, we will learn how to integrate this demo with some of the on-board components.

### Making Local Actuation Happen

The Texas Instruments CC3220SF LaunchPad is a feature-packed development board with a lot of options.  When you look at all the LEDs, buttons, and pins -- each of these are configurable in their own way through what is called Pin Multiplexing.

Out of the box, the LEDs are disabled because they are shared on the same bus as I2C and PWM peripherals.  The on-board sensors, which we will use in Lab 3 and Lab 4, conflict with the LEDs.  In this lab, we will enable the GPIO but disable them in the next lab.

1. In CCS8, open ```application_code``` > ```ti_code``` > ```CC3220SF_LAUNCHXL.c```.
2. Scroll to line 225, for pin configuration ```GPIO_PinConfig gpioPinConfigs[]```.
3. The changes are done to the last four elements in the array, so the array will look like the following.  Note the commas aded to the unremarked lines.

   ```c
    CC3220SF_LAUNCHXL_GPIO_LED_D6,
    /*GPIOCC32XX_GPIO_10 | GPIO_CFG_OUT_STD | GPIO_CFG_OUT_STR_HIGH | GPIO_CFG_OUT_LOW,*/
    CC3220SF_LAUNCHXL_GPIO_LED_D5,
    /* GPIOCC32XX_GPIO_11 | GPIO_CFG_OUT_STD | GPIO_CFG_OUT_STR_HIGH | GPIO_CFG_OUT_LOW,*/
   ```
4. In CCS8, open ```application_code``` > ```main.c```.  Scroll to line 126.  Note the API call ```GPIO_Init()```.  This API call is not part of Amazon FreeRTOS; rather, it is part of the TI SDK.  Since it is already being initialized by the scheduler startup hook, we do not need to add it again.

Next, we will add GPIO code to perform a tick-tock action between the two GPIOs.

1. In CCS8, open ```application_code``` > ```common_demos``` > ```source``` > ```aws_hello_world.c```.
2. Scroll to approximately line 510, to function ```void vStartMQTTEchoDemo( void )```.

   ```c
       configPRINTF( ( "Creating MQTT Echo Task...\r\n" ) );

       GPIO_write(Board_GPIO_LED1, Board_GPIO_LED_OFF);
       GPIO_write(Board_GPIO_LED2, Board_GPIO_LED_OFF);

       prvMacForHumans(thing_mac_address);

       /* Create the message buffer used to pass strings from the MQTT callback
   ```
3. Next, we will turn on one of the LEDs in the publish function.  Find the function ```static void prvPublishNextMessage( BaseType_t xMessageNumber )```.  Add the following directly before the ```( void ) xReturned``` statement.

   ```c
        GPIO_write(Board_GPIO_LED2, Board_GPIO_LED_OFF);
        GPIO_write(Board_GPIO_LED1, Board_GPIO_LED_ON);
   ```
4. Next, we will add the opposite effect to the task performing the ACK publish.  The function name is ```static void prvMessageEchoingTask( void * pvParameters )```.  At the end of the block within the following ```if``` condition, place the alternating LED code.

   ```
           if( xBytesReceived <= ( sizeof( cDataBuffer ) - ( size_t ) echoACK_STRING_LENGTH ) )
           {
               ...
               GPIO_write(Board_GPIO_LED1, Board_GPIO_LED_OFF);
               GPIO_write(Board_GPIO_LED2, Board_GPIO_LED_ON);
           }
   ```
5. Run the CCS project again.  You will notice the led alternating on what is marked D8 and D9 on the development board.

[Next Lab](./Lab3.md)
