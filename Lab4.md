# Amazon FreeRTOS Workshop for the Texas Instruments CC3220SF
## Lab 4: Using Device Shadows

In this lab, you will program Amazon FreeRTOS to change the on-board LED state based on accelerometer readings reaching a specific threshold. The standard way to perform this actuation through AWS IoT Core is by using Device Shadows. You will learn the details about Reported and Desired state and understand how to drive signals across Amazon FreeRTOS tasks using Queues.


### Premise

In this lab, you will discover the importance of the AWS IoT Device Shadow and how it enables command and control cabilities for your solution.

Command and control is the act of telling the physical entity to achieve a potential, and then the autonomous entity, through programmatic means, achieves that potential.

More specifically, you want a device to perform a specific act.  The communication must be explicit and prescriptive.  The act must be transcribed to a language the edge device understands.  The edge device must know how to receive the communication and transcribe it to instructions that eventually affect the environment.

Likewise, the human prescribing this act wants to know when, and sometimes to what degree, the act has been achieved. It is simpler if the channel under which the two entities communicate use the same language, but obviously use different contexts.
      
This is the crux of device shadow.  *Active* communication under different contexts as well as abrupt communication under specific contexts (such as initialization when context or current state must be understood).

This lab gets you started on Device Shadows.  Although we are not doing any actuation, we will update the device shadow, you will see changes in the Console, and perform some action in the AWS Cloud.

### Start the Project

Within the Github project, there is the directory ```lab4/start```.  This is the project where we will be starting the lab.

1. Open Code Composer Studio (CCS).
2. Open **File** > **Import...**
3. Select **CCS Projects**.  Click **Next**.
4. In the Select search-directory radio button item, choose **Browse...**.  Choose ```LAB_REPOPATH/Lab4/start```.


### Organizing the Accelerometer Task

As previously mentioned, tasks help you organize and manage your code in an encapsulated and reusable way.

The accelerometer is a sensor component on the CC3220SF development board.  The accelerometer communicates with the microcontroller over the I2C bus.  The accelerometer is a BNO222 manufactured by Bosch.

The challenging part about our next task is now we will be communicating to two different components on the same I2C bus.  If both components communicate at the same time, there will be a conflict.

The way to manage this is by ensuring the communication -- the send and receive -- is processed by the microcontroller in what can be considered as similar to a transaction.  To ensure the send and receive can be achieved without corruption we need exclusive access.  The mechanism for handling exclusive access is with a Mutex: short for mutual exclusion.

#### Organizing Dependencies

The accelerometer task uses some helper libraries that are delivered as part of the Texas Instruments CC3220SF SDK.  These files are organized in the repository under ```LAB_REPOPATH/extern```.  We need to include these files into CCS and add a project include path for the compilation process.

1. Add the files to the build.
   1. Right-click the project root, and select New > Folder.
   2. For Folder name enter **extern**.
   3. Click the Advanced button.
   4. Click the radio button with the label **Link to alternate location (Linked Folder)**.
   5. Click the Browse button.
   6. Locate the directory ```LAB_REPOPATH/extern```.
   7. Click OK.
   8. Click Finish.
9. Next, we need to add the include path for the ```extern/include``` directory.
   1. Right-click the project root, and select Properties.
   2. Select Build > ARM Compiler > Include Options.  Scroll to the bottom, select the last element, and then click the icon that looks like a document with a green plus sign on it.
   3. Enter the following value:

      ```c
      ${workspace_loc:/${ProjName}/extern}
      ```
      And click OK.
   4. Click Apply and Close.

#### Initialize the Mutex Semaphore

In the next section, we will be implementing the Accelerometer code. The Mutex conditions will be taking place. Let's initialize the Mutex beforehand.

1. Declare the mutex variable.  You can place it after the I2C handle declaratation around line 195.

	```c
	SemaphoreHandle_t xSemaphore;
	```
2. Initialize the semaphore. This will be done in ```void vStartMQTTEchoDemo( void )```.  You can place it after the ```xMessageBuffer``` assertion.  It just needs to happen before the Temperature and Accelerometer tasks start.

   ```c
       xSemaphore = xSemaphoreCreateMutex();
   ```

#### The Accelerometer Task

We will start off by implementing the accelerometer task.  We will first need to setup a declaration for it, and then go into the details of the code itself.

1. Start by adding the function stub.

   ```c
   static void prvAccSensorReaderTask( void * pvParameters )
   {
   }
   ```
   The next steps will be in the context of this function.
2. Add the declarations to be used throughout the code.

   ```c
	    char cDataBuffer[ 128 ];
	    xMQTTTaskParameter *pxParameters;
	    pxParameters = ( xMQTTTaskParameter * ) pvParameters;
	
	    signed char accX = 0;
	    signed char accY = 0;
	    signed char accZ = 0;
	
	    signed char offset_accX = 0;
	    signed char offset_accY = 0;
	    signed char offset_accZ = 0;
   ```
3. Open the BMA222.  This API function is defined in the source code we added under ```extern```.  We will wrap this in an ```xSemaphoreTake``` call to ensure we have control of the bus.

    ```c
	    if ( xSemaphoreTake( xSemaphore, 10 ) == pdTRUE )
	    {
	        BMA222Open( xI2C );
	        xSemaphoreGive( xSemaphore );
	    }
    ```
4. Block on waiting on semaphore to do the calibration.  Read the values and then give back the semaphore.
   
   ```c
       if ( xSemaphoreTake( xSemaphore, 10 ) == pdTRUE )
	    {
	        if ( BMA222Read( xI2C, &accX, &accY, &accZ ) == 0 )
	        {
	            offset_accX = accX;
	            offset_accY = accY;
	            offset_accZ = accZ;
	            xSemaphoreGive( xSemaphore );
	        }
	        else
	        {
	            configPRINTF(("BMA222 read failed!"));
	            xSemaphoreGive( xSemaphore );
	            while (0);
	        }
	    }
    ```

The read-report process will run within an endless while loop.

1. Add the start of the while loop.  Ensure we get information from the sensor by getting the Mutex representing access to the I2C bus.

   ```c
       while (1)
       {
           if ( xSemaphoreTake( xSemaphore, 10 ) == pdTRUE )
           {
               if ( BMA222Read( xI2C, &accX, &accY, &accZ ) == 0 )
               {
                   xSemaphoreGive( xSemaphore );
   ```
2. Next, calculate the offset from the original calibrated location.  These values will be used to report information to the device shadow.

   ```c
                   accX = accX - offset_accX;
                   accY = accY - offset_accY;
                   accZ = accZ - offset_accZ;
   ```
3. Construct the JSON payload that will be sent to the AWS Cloud.

   ```c
                   snprintf(cDataBuffer, sizeof( cDataBuffer),
                                      "{"
                                      "\"state\":{"
                                      "\"reported\":{"
                                      "\"x\":%d,"
                                      "\"y\":%d,"
                                      "\"z\":%d"
                                      "}},"
                                      "\"clientToken\": \"token-%d\"" \
                                      "}",
                                      accX, accY, accZ,
                                      ( int ) xTaskGetTickCount() );
   ```

	When it reaches the AWS Cloud, it will look similar to the following.  **DO NOT ADD THIS TO THE C SOURCE FILE**
	
	```json
	{
	  "state": {
	    "reported": {
	      "x": 0,
	      "y": -4,
	      "z": 65
	    }
	  },
	  "clientToken": "token-2218795"
	}
	```
4. Add the code that sends the payload to the AWS Cloud. Pay special attention to the **topic** where the payload is being sent.  This is configured with the xCreateTask call we will discuss in the next session.  The publishing is to the special ```$aws``` topic specifically for the Client ID named with the MAC address.

   ```c
                configPRINTF(("##### accX: %d, accY: %d, accZ: %d. #####\r\n", accX, accY, accZ));
                MQTTAgentPublishParams_t pxPublishParams;
                pxPublishParams.pucTopic = ( uint8_t * ) pxParameters->topic;
                pxPublishParams.usTopicLength = ( uint16_t )  strlen(( const char * )pxParameters->topic);
                pxPublishParams.pvData = cDataBuffer;
                pxPublishParams.ulDataLength = ( uint32_t )  strlen(( const char * )cDataBuffer);
                pxPublishParams.xQoS =  eMQTTQoS1;

                if ( MQTT_AGENT_Publish(xMQTTHandle, &( pxPublishParams ),
                        democonfigMQTT_TIMEOUT) == eMQTTSuccess )
                {
                    configPRINTF(("Outbound sent successfully.\r\n"));
                }
                else
                {
                    configPRINTF(("Outbound NOT sent successfully.\r\n"));
                }
            }
        }

        vTaskDelay( pdMS_TO_TICKS( TEMPERATURE_TASK_READ_DELAY_MS ) );
   ```

#### Adding xCreateTask for the Accelerometer

Before doing this, you might have asked yourself: don't we need to initialize I2C like we did in the previous lab?  The answer is No.  However, we need to "pull up" the initialization so it's done only once for both Tasks.  We will pull up that code first, and then implement the xCreateTask call.

1. Pull up code from the Temperature task to the ```void vStartMQTTEchoDemo( void )``` function.  Cut and paste the following code.  Ensure this code is removed from the Temperature task.  Although it will unlikely harm due to the conditions, there is no need for the additional instructions.  In ```void vStartMQTTEchoDemo( void )```, you can place this code directly before the ```xSemaphoreCreateMutex``` call.

   ```c
    /*
     *  Create/Open the I2C bus that talks to the TMP sensor
     */
    I2C_Params      i2cParams;
    I2C_Params_init(&i2cParams);
    i2cParams.bitRate = I2C_400kHz;

    if ( xI2C == NULL )
      xI2C = I2C_open( Board_I2C_TMP,  // Defined in Board.h
                       &i2cParams );
    vTaskDelay( pdMS_TO_TICKS( 7 ) ); // ensure configuration mode completes, 7ms, super magic number

    if ( xI2C == NULL ) {
      configPRINTF(("CRITICAL: Could not open I2C bus. Reset the board.\r\n"));
      while (1);
    }
   ```
2. Next, add the declaration of the properties structure we will use to compose in the topic name for the accelerometer.  You can put this directly under the declaration for the properties we did in the previous lab.

   ```c
       xMQTTTaskParameter taskParameter_accelerometer; /* Lab 4 */
   ```
3. Finally, we add the call to xTaskCreate.  Directly after the similar block for the Temperature sensor, add this code. Pay special attention to the topic name.  This is the topic where shadow updates get published.

   ```c
    if ( xReturned == pdPASS )
    {
        memset( taskParameter_accelerometer.topic, 0x00, sizeof( taskParameter_accelerometer.topic ) );
        snprintf( taskParameter_accelerometer.topic, sizeof( taskParameter_accelerometer.topic ), "$aws/things/%s/shadow/update", thing_mac_address );

        xReturned = xTaskCreate( prvAccSensorReaderTask,
                                "BMA222",
                                TEMP_TASK_STACK_SIZE,
                                ( void * ) &taskParameter_accelerometer,
                                TEMP_TASK_PRIORITY,
                                NULL );
    }
   ```

#### Adding Semaphore Check to Temperature Task

The last step to getting our code to work with two sensors on the same I2C bus is to also have the SemaphoreTake-SemaphoreGive pair for the API calls within the Task.

After the fourth line in the ```while(1)``` loop, wrap the I2C_transfer call.  After wrapping, it should look like this.

```c
        if ( xSemaphoreTake( xSemaphore, 10 ) == pdTRUE )
        {
            /* Do not hold the semaphore for longer than needed. */
            if ( I2C_transfer(xI2C, &i2cTxn) ) {
                temperatureC = (rxBuf[0] << 6) | (rxBuf[1] >> 2);
            }
            xSemaphoreGive( xSemaphore );
        }
```

Why is the ```xSemaphoreGive``` call not directly beneath the instruction for manipulating the 

#### Running the code

Now, we will run the code from Code Composer Studio.  Do a full build, and then begin the debug process.

What you will see in the terminal output window will be interesting - you will note that messages saying "Outbound sent successfully" are virtually right next to each other.

```text
46 29150 [BMA222] accX: -14, accY: 0, accZ: 1
47 29151 [TMP006] Celsius: 27.500000, Fahrenheit: 81.500000
49 29254 [BMA222] Outbound sent successfully.
48 29254 [TMP006] Outbound sent successfully.
```

As you can see, the two tasks seem to be running simultaneously.  Since this code is running on a single core, it's not multithreaded but instruction execution is being handled by the scheduler.  Since both the BMA222 and TMP006 tasks are running at the same priority (the default priority -- we didn't change it), the scheduler is handling the "balance".

What would happen if we, for example, changed the ```xSemaphoreTake``` in the BMA222 routine to cover virtually the whole task code block?  Certainly, we would not get this sharing effect, even when the tasks have the same priority.  This would cause BMA222 to "take up most of the time", making it seem to have higher priority due to it acquiring the Semaphore (i.e. acquiring the lock).

### Experiencing the Cloud Integration

In this section, you will send an alert to your mobile phone when your board's reported state exceeds the desired state.
