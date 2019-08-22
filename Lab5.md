# Amazon FreeRTOS Workshop for the Texas Instruments CC3220SF
## Lab 5: Over the Air Updates

In this project, you'll learn how to deploy an over-the-air (OTA) update to a microcontroller (MCU) running Amazon FreeRTOS.  You will need to use a development board or kit with an Amazon FreeRTOS qualified MCU in order to deploy the OTA in a developer friendly setting. In this tutorial, you will use the Texas Instruments SimpleLink™ Wi-Fi® CC3220SF LaunchPad™ development kit (CC3220SF-LAUNCHXL). This project will give you the tools and experience you need to build OTA functionality into your own devices that use qualified MCUs.

### Create Code Signing Certificates


1. Create a Code-Signing Certificate and Private Key

   To digitally sign firmware images, you need a code-signing certificate and private key. For testing purposes, you can create a self‐signed certificate and private key. For production environments, purchase a certificate through a well‐known certificate authority (CA)
 
2. Creating Code-Signing Certificates for Texas Instruments CC3220SF-LAUNCHX

   The SimpleLink Wi-Fi CC3220SF Wireless Microcontroller Launchpad Development Kit supports two certificate chains for firmware code signing: 

   * Production  (certificate-catalog)
  
     To use the production certificate chain, you must purchase a commercial code-signing certificate and use the TI Uniflash tool (http://www.ti.com/tool/UNIFLASH) to set the board to production mode. 

   * Testing  and development (certificate-playground)
  
     The playground certificate chain allows you to try out OTA updates with a self‐signed code-signing certificate. 

  > Windows
  > > The location of the certificates are in 
  > > ```SIMPLELINK_SDK_BASE\tools\cc32xx_tools\certificate-playground```
  > > Where ```SIMPLELINK_SDK_BASE``` is usually ```C:\ti\simplelink_cc13x2_sdk_2_10_00_48```
  > 
  > MacOS and Linux
  > > The location of the certificates are in 
  > > ```SIMPLELINK_SDK_BASE/tools/cc32xx_tools/certificate-playground```
  > > Where ```SIMPLELINK_SDK_BASE``` is usually ```/Applications/ti/simplelink_cc13x2_sdk_2_10_00_48```

  The certificates in the SimpleLink CC3220 SDK are in DER format. You need to convert them to PEM format to create a self‐signed code-signing certificate. 

  Follow these steps to create a code-signing certificate that is linked to the Texas   Instruments playground certificate hierarchy and meets AWS Certificate Manager and Code Signing for Amazon FreeRTOS criteria. 
 
3. Create a **self**‐**signed** code signing certificate

   In your working directory, ```LAB_REPOPATH/credentials```,use the following text to create a file named *cert_config*. Replace test_signer@amazon.com  with your email address. 

   ```
   [ req ]
   prompt             = no
   distinguished_name = my dn
 
   [ my dn ]
   commonName = test_signer@amazon.com
 
   [ my_exts ]
   keyUsage         = digitalSignature
   extendedKeyUsage = codeSigning

   ```

4. Create  a private key and certificate signing request (CSR):

   ```bash
   openssl req -config cert_config  \
                -extensions my_exts  \
                -nodes               \
                -days 365            \
                -newkey rsa:2048.    \
                -keyout tisigner.key \
                -out tisigner.csr
   ```

   Convert  the Texas Instruments playground root CA *private key* from DER format to  PEM format. 

   The TI playground root CA *private key* is located in:


   > Windows
   > > ```SIMPLELINK_SDK_BASE*\tools\cc32xx_tools\certificate-playground\dummy-root-ca-cert-key```
   >
   > Mac OSX
   > > ```SIMPLELINK_SDK_BASE*/tools/cc32xx_tools/certificate-playground/dummy-root-ca-cert-key```

   Run the command below to perform the conversion.

   ```bash
   cp $SIMPLELINK_SDK_BASE/tools/cc32xx_tools/certificate-playground/dummy-root-ca-cert-key .
   openssl rsa -inform DER -in dummy-root-ca-cert-key -out dummy-root-ca-cert-key.pem
   ```
   
   Convert  the Texas Instruments playground *root CA* certificate from DER format to  PEM format. 

   The TI playground root CA certificate is located here:

   > Windows
   > > ```SIMPLELINK_SDK_BASE\tools\cc32xx_tools\certificate-playground/dummy-root-ca-cert```
   >
   > Mac OSX
   > > ```SIMPLELINK_SDK_BASE/tools/cc32xx_tools/certificate-playground/dummy-root-ca-cert```

   Run the command below to perform the conversion.

   ```bash
   cp $SIMPLELINK_SDK_BASE/tools/cc32xx_tools/certificate-playground/dummy-root-ca-cert .
   openssl x509 -inform DER -in dummy-root-ca-cert -out dummy-root-ca-cert.pem
   ```
   
   Sign  the CSR with the Texas Instruments *root CA*:

   ```bash
   openssl x509 -extfile cert_config                \
                -extensions my_exts                  \
                -req -days 365                       \
                -in tisigner.csr                     \
                -CA dummy-root-ca-cert.pem           \
                -CAkey dummy-root-ca-cert-key.pem    \
                -set_serial 01                       \
                -out tisigner.crt.pem                \
                -sha1
   ```

   Convert  your code-signing certificate (tisigner.crt.pem) to DER format: 

   ```bash
   openssl x509 -in tisigner.crt.pem \
                -out tisigner.crt.der \
                -outform DER
   ```
   
   Note: You write the tisigner.crt.der certificate onto the TI development board later. 

### Import the Certificates into AWS Certificate Manager

Once you generated the certificates from the openssl, you can import them into AWS Certificate Manager to be used by AWS IoT Core.

1. Sign in to the https://console.aws.amazon.com/acm
2. Under the Provision certificates, click on the *Get started*
3. On the top of the screen, click on the **Import a certificate**
4. Use a text editor, open up the file ```tisigner.crt.pem```
5. Copy the whole contents from the file, with the complete ```-----BEGIN CERTIFICATE-----``` and ```-----END CERTIFICATE-----``` lines, and paste the contents in the text area of **Certificate body**, and close the text editor.
6. Use a text editor, open up the file tisigner.key
7. Copy the whole contents from the file, with the complete ```-----BEGIN RSA PRIVATE KEY-----``` and ```-----END RSA PRIVATE KEY-----``` lines, and paste the contents in the text area of **Certificate private key**, and close the text editor.
8. Use a text editor, open up the file dummy-root-ca-cert.pem
9. Copy the whole contents from the file, with the complete ```-----BEGIN CERTIFICATE-----``` and ```-----END CERTIFICATE-----``` lines, and paste the contents in the text area of **Certificate chain**, and close the text editor.
10. Click on the “**Review and import**” at the bottom
11. Take a quick look at the contents, and click **Import**
12. Take note of the ARN in the Details section. You will need the ARN of the certificate in the later when you setup the FreeRTOS OTA job.

For future reference, you can also use the command-line:

```bash
aws acm import-certificate --certificate file://tisigner.crt.pem      \
                           --private-key file://tisigner.key          \
                           --certificate-chain file://dummy-root-ca-cert.pem
```

### Add the OTA Update Demo

Next, we need to add OTA Update code to our application. In this lab, we will use the OTA Update demo code that Amazon FreeRTOS provides.  This means that it's only a two step process to add the capability to our project.

1. Add the include file to ```aws_hello_world.c```.  Locate the following on approximately line 81.

   ```c
   #include "aws_hello_world.h"
   ```
   
   And add the ota demo include file directly below it.
   
   ```c
   #include "aws_ota_update_demo.h"
   ```
2. Add the function call to start the demo.  At the end of the ```vStartMQTTEchoDemo( void )``` function, add this line.

   ```c
   vStartOTAUpdateDemoTask();
   ```
   
That's it! Now we are prepared to start our update.


### Flash the Amazon FreeRTOS demo application

*To burn the demo application onto your board*

1. On your Texas Instruments developer board, place the SOP jumper on the middle set  of pins (position = 1) and reset the board. 
2. Switch to the  Uniflash application (last used in Lab 1).
5. Disconnect  your terminal emulator and choose the *Connect* button on the right  side of the Uniflash application window. 
6. On the  left, under **Files**, choose **User Files**. 
7. In the *File*  selector pane, choose the *Add File* icon . 
8. Browse  to the ```SIMPLELINK_SDK_BASE/tools/cc32xx_tools/certificate-playground```  directory, select ```dummy-root-ca-cert```,  choose **Open**, and then choose **Write**. 
9. Browse  to the working directory where you created the code-signing certificate  and private key, choose tisigner.crt.der,  choose **Open**, and then choose **Write**. 
10. From  the **Action** drop-down list, choose **Select MCU Image**, and then  choose **Browse** to choose the firmware image to use write to your  device (**aws_demos.bin**). This file is located in the AmazonFreeRTOS/demos/ti/cc3200_launchpad/ccs/Debug  directory. Choose **Open**.
11. In the  file dialog box, confirm the file name is set to ```mcuflashimg.bin```.  Select the *Vendor* check box. Under *File Token*, type ```1952007250```. Under *Private Key File  Name*, choose *Browse* and then choose tisignerkey from the working directory  where you created the code-signing certificate and private key. Under *Certification  File Name*, choose tisigner.crt.der,  and then choose *Write*. 
12. In the  left pane, under *Files*, select *Trusted Root-Certificate Catalog*.  
13. Clear  the *Use default Trusted Root-Certificate Catalog* check box. 
14. Under *Source  File*, choose *Browse*, select *simplelink_cc32xx_sdk_1_40_01_00/tools/certificate-playground\certcatalogPlayGround20160911.lst*,  and then choose *Open*. 
15. Under *Signature  Source File*, choose *Browse*, select *simplelink_cc32xx_sdk_1_40_01_00/tools/certificate-playground\certcatalogPlayGround20160911.lst.signed.bin*,  and then choose *Open*. 
16. Choose  the SAVE (get button image from https://docs.aws.amazon.com/freertos/latest/userguide/images/save.png ) button to save your project.\
17. Choose  the BURN (get button image from https://docs.aws.amazon.com/freertos/latest/userguide/images/flame.png ) button.
18. Choose *Program  Image (Create and Program)*. 
19. After  programming process is complete, place the SOP jumper onto the first set  of pins (position = 0), reset the board, and reconnect your terminal  emulator to make sure the output is the same as when you debugged the demo  with Code Composer Studio. 

### Deploy an Amazon FreeRTOS OTA

In this module, we will make use of the work completed in the prior modules to build and deploy an updated firmware image using the OTA features of AWS IoT and AWS FreeRTOS.

Time to Complete: 30 min
Technologies Used: TI Code Composer Studio, Amazon S3, AWS IoT Jobs, AWS IAM, AWS FreeRTOS OTA Agent

Architecture Overview




AWS IoT Device Management allows you to schedule Amazon FreeRTOS device software updates with the Amazon FreeRTOS over-the-air (OTA) update job. An AWS IoT job is used to define a set of remote operations that are sent to and executed on one or more devices connected to AWS IoT.

Key concepts of AWS IoT jobs:

* target
    When you create a job, you specify a list of targets which are the devices that should perform the operations. The targets can be things, thing groups, or both. AWS IoT Jobs sends a message to each target to inform it that a job is available.
* job document
    Job document is a UTF-8 encoded JSON document that describes the remote operations to be performed by the targets.For an OTA job, the job document will contain one or more URLs where the target can download an update or some other data, and any code signing specifications if required.
    For Amazon FreeRTOS OTA job, there are 3 options for code signing:
    * *Sign a new firmware image for me*
        The firmware image will be signed by the code signer service for Amazon FreeRTOS feature. Code signing provides cryptographic signing of an image file to ensure only genuine code can be installed and run on Amazon FreeRTOS devices.
        Once the firmware is signed, there will be 2 formats of the firmware in the S3: the original uploaded firmware image, and the signed firmware image artifact. The signed firmware image artifact will be stored in the folder “SignedImages” in the same S3 bucket as the original firmware image.
        *This is the option we will be using in this module*
    * *Select a previously signed firmware image*
        You can reuse a previous code signed image.
    * *Use my custom signed firmware image*
        If you have specific security requirements, you may choose to use your own code signer and provide the file signature and algorithm with this option.
* job type
    A job can be either a snapshot job, or a continuous job. 
    A *snapshot* job will be sent to targets selected, and after those targets complete the job (or report that they are unable to do so), the job will be in a complete state.
    *This is the option we will be using in this module*
    A *continuous* job is one that continues to run and is executed when a change is detected in a target. For example, a job will run on a device when the thing representing the device is added to a target group, even after the job was completed by all things originally in the group. A continuous job can be used to onboard or upgrade devices as they are added to a group. 
* job execution
    A job execution is an instance of a job on a target device. The target starts an execution of a job by downloading the job document. It then performs the operations the document specifies, and reports its progress to AWS IoT. An execution number is a unique identifier of a specific job execution on a specific target. 
    When you schedule an OTA job, you have to enter a unique ID for each OTA job execution.

Once an OTA job is created in AWS IoT Device Management, the AWS IoT Device Management will publish an OTA job message via the MQTT, and the OTA Library and Agent running in the Amazon FreeRTOS sample application on the device subscribing to the topic will start streaming down the signed firmware image in chunks via the MQTT.

Implementation Instructions

### Create an Amazon S3 Bucket to Store a Firmware Image 

Updated firmware will be upload to a Amazon S3 bucket.
 
To create an Amazon S3 bucket
1.         Go to the *https://console.aws.amazon.com/s3/*. 
2.         Choose *Create bucket*. 
3.         On the Bucket name, type a bucket name, and then choose *Next*. 
4.         On the Create bucket page, choose *Versioning*. 
5.         Choose *Enable versioning*, and then choose *Save*. 
6.         Choose *Next*. 
7.         Choose *Next* to accept the default permissions. 
8.         Choose *Create bucket*.  

Step 2: Set up the OTA Update Service Role 

The following are required to perform OTA updates:

* *An OTA service role.
    *The OTA Update service assumes this role to create and manage OTA update jobs on your behalf.
* *An OTA user policy.
    *This policy grants your IAM user access to a number of services required for OTA updates. 
* *A code signing policy.
    *The code signing policy grants your IAM user access to the Code Signing for Amazon FreeRTOS service.

To create an OTA service role

1. Sign in to the https://console.aws.amazon.com/iam/.
2. From the left navigation pane, choose *Roles*.
3. Choose the *Create role* button.
4.  Under Select type of trusted entity, choose *AWS service*.
5.  Choose *IoT* from the list of AWS services.
6.  Choose *Next: Permissions*.
7.  Choose *Next: Review*.
8.  Type a role name and description, and then choose *Create role*. 

 
To add OTA update permissions to your OTA service role

1. In the *search box* on the left navigation pane of the IAM console page, type your role's name and then choose it from the list.
2.  Choose the *Attach policy* button.
3.  In the Search box, type *AmazonFreeRTOSUpdatex*, choose it in the list of managed policies, and then choose the *Attach policy* button. 

 
To add Amazon S3 permissions to your OTA service role

1.    In the *search box* on the left navigation pane of the IAM console page, type your role's name and then choose it from the list. 
2.    In the lower right, choose *Add inline policy*.
3.    Choose the *JSON* tab.
4.    Copy and paste the following policy document into the text box. Replace <example-bucket> with the name of your bucket. 

{
     "Version": "2012-10-17",
     "Statement": [
     {
          "Effect": "Allow",
          "Action": [
               "s3:GetObjectVersion",
               "s3:GetObject",
               "s3:PutObject"
          ],
          "Resource": "arn:aws:s3:::<example-bucket>/*"
     }
     ]
}

5.    Choose *Review policy*.
6.    Type a name for the policy and choose *Create policy*. 

To add the required IAM permissions to your OTA service role

1. In the search box on the IAM console page, enter the name of your role, and then choose it from the list.
2. Choose Add inline policy.
3. Choose the JSON tab.
4. Copy and paste the following policy document into the text box:

{
    "Version": "2012-10-17",
    "Statement": [
      {
            "Effect": "Allow",
            "Action": [
                "iam:GetRole",
                "iam:PassRole"
            ],
            "Resource": "arn:aws:iam::<your_account_id>:role/<your_role_name>"
      }
    ]
}

Make sure that you replace <your_account_id> with your AWS account ID, and <your_role_name> with the name of the OTA service role.

5. Choose Review policy.
6. Enter a name for the policy, and then choose Create policy.

Creating an OTA User Policy 

You must grant your IAM user permission to perform over-the-air updates. Your IAM user must have permissions to: 
 
•          Access the S3 bucket where your firmware updates are stored.
•          Access certificates stored in AWS Certificate Manager.
•          Access the Amazon streaming service. 
•          Access the Amazon FreeRTOS OTA Update service.
•          Access the AWS IoT Jobs service.
•          Access IAM.
•          Access the Code Signing for Amazon FreeRTOS service.
 
Grant your IAM user the required permissions by creating an OTA user policy and attaching it to your IAM user. 
 
To create an OTA user policy
 
1.         Open the https://console.aws.amazon.com/iam/ console. 
2.         From the left navigation pane, choose *Users*. 
3.         Choose your IAM user from the list.
4.         Choose *Add permissions*. 
5.         Choose *Attach existing policies directly*. 
6.         Choose the *Create policy* button. 
7.         On the *JSON* tab, copy and paste the following policy document into the policy editor. Replace <example-bucket> with the name of the Amazon S3 bucket where you store your OTA update firmware image. Replace <your-account-id> with your AWS account ID. You can find your AWS account ID in the upper-right corner of the console. When you enter your account ID, remove any dashes (-). Replace <role-name> with the name of the IAM service role you just created. 

```json
{
      "Version":"2012-10-17",
      "Statement":[
      {
            "Effect":"Allow",
            "Action":[
                  "s3:ListBucket",
                  "s3:ListAllMyBuckets",
                  "s3:CreateBucket",
                  "s3:PutBucketVersioning",
                  "s3:GetBucketLocation",
                  "s3:GetObjectVersion",
                  "acm:ImportCertificate",
                  "acm:ListCertificates",
                  "iot:ListThings",
                  "iot:ListThingGroups",
                  "iot:CreateStream",
                  "iot:CreateOTAUpdate",
                  "iot:GetOTAUpdate",
                  "iot:ListJobs",
                  "iot:ListJobExecutionsForJob",
                  "iot:DescribeJob",
                  "iot:GetJobDocument",
                  "iam:ListRoles",
                  "signer:ListSigningJobs",
                  "signer:StartSigningJob",
                  "signer:DescribeSigningJob"
            ],
            "Resource":"*"
      },
      {
      "Effect":"Allow",
      "Action":[
            "s3:GetObject",
            "s3:PutObject"
      ],
      "Resource":"arn:aws:s3:::<example-bucket>/*"
      },
      {
            "Effect":"Allow",
            "Action":"iam:PassRole",
            "Resource":"arn:aws:iam::<your-account-id>:role/<role-name>"
      }
      ]
}
```

8.         Choose the *Review policy* button. 
9.         Type a name for your new OTA user policy, and then choose the *Create policy* button. 
 
To attach the OTA user policy to your IAM user

1.         Open the https://console.aws.amazon.com/iam/ console. 
2.         From the left navigation pane, choose *Users*. 
3.         Choose your IAM user from the list.
2.         Choose *Add permissions*. 
3.         Choose *Attach existing policies directly*. 
4.         Search for the OTA user policy you just created and select the *check box* next to it. 
5.         Choose the *Next: Review* button. 
6.         Choose the *Add permissions* button. 

Granting Access to the Code Signing Service 

In production environments, you should digitally sign your firmware update to ensure the authenticity and integrity of the update. You can sign your update manually or use the Code Signing for Amazon FreeRTOS service in the AWS IoT console to sign your code. To sign your code automatically, you must grant your AWS account access to the Code Signing for Amazon FreeRTOS service. 
 
To grant your AWS account access to Code Signing for Amazon FreeRTOS
 
1. Sign in to the https://console.aws.amazon.com/iam/. 
2. From the left navigation pane, choose *Policies*.
3. Choose *Create Policy*. 
4. On the *JSON* tab, copy and paste the following JSON document into the policy editor. This policy allows the IAM user access to all code signing operations. 

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "signer:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```
5. Choose *Review policy*. 
6. Type a policy name and description, and then choose *Create policy*. 
7. In the navigation pane, choose *Users*. 
8. Choose your IAM user account.
9. On the Permissions tab, choose *Add permissions*. 
10. Choose **Attach existing policies directly**, and then select the **check box** next to the code signing policy you just created. 
11. Choose **Next: Review**. 
12. Choose **Add permissions**. 

### Creating a Firmware Update

Now you can create an updated firmware image to install using the OTA Update service. In these steps, we just update the firmware version of the OTA demo application. Because the OTA demo displays the version of the firmware on device reset, we can use the displayed version to verify the update was successful. 
 
To create an updated version of the firmware
 
1. Open the aws_demos project in Code Composer Studio.
2. Open ```aws_demos/application_code/common_demos/include/aws_application_version.h``` and set ```APP_VERSION_BUILD``` to ```1```. See example in screen shot.
3. Rebuild the aws_demos project. 

###Uploading Updated Firmware 

The Amazon FreeRTOS OTA Update service loads your updated firmware from an Amazon S3 bucket. 
 
To upload your firmware image to an Amazon S3 bucket

1. Go to the [Amazon S3 console](https://console.aws.amazon.com/s3/).
2. In Search for buckets, type your bucket name for the bucket you created in *Create an Amazon S3 Bucket to Store a Firmware Image*. 
3. Choose **Upload**.
4. Choose **Add files**, choose the firmware image you want to upload. The firmware image file will have a .bin extension for example aws_demo.bin, and then choose **Next**.
5. Choose **Next**, and then choose **Next** again.
6. Choose **Upload**. 

### Create an OTA Update Job 

At this point, the OTA agent will be running in the TI CC3220SF-LAUNCHXL. Below is the basic flow of *aws_demos*. 

[OTA_AgentInit()](https://docs.aws.amazon.com/freertos/latest/lib-ref/aws__ts__ota__agent_8h.html#a05967623d9cb8c23ed3e01471a3d5844) is the main function that subscribe to MQTT message for new OTA job. Once a new message is received, the OTA agent will stream the new image from the AWS IoT Device Management. Once the OTA job is completed, it will callback to the *App_OTACompleteCallback()* defined in *aws_ota_update_demo.c*

At the AWS side, you have uploaded the new firmware image onto the S3 bucket. In this step, you will trigger the OTA process with an OTA update job.

1.    Open the https://console.aws.amazon.com/iot/ console.
2.    From the left navigation pane, choose *Manage*, and then choose *Jobs*.
3.    Choose *Create a job*. 
4.    Under Create an Amazon FreeRTOS OTA update job, choose *Create OTA update job*. 
5.    Under Select devices to update, choose *Select*, and then choose the *Things* tab in the device list.
6.    Select the *check box* next to the IoT thing associated with your CC3220SF-LAUNCHXL board.
7.    Under Select and sign your firmware image, choose *Sign a new firmware image for me*. 
8.    In the drop down list of Device hardware platform, select *CC3220SF-LAUNCHXL*
9.*    *Under Pathname of code signing certificate on device, type tisigner.crt.der. 
10.  Make sure *Pathname of firmware image on device* is set to /sys/mcuflashimg.bin
11.  Under Select your firmware image in S3, choose *Select*, and then choose the aws_demos.bin image that contains your updated image.
12.  Under Code signing certificate, choose *Select*, and then choose your code-signing certificate. Code-signing certificates are listed by ARN. The ARN for your certificate was displayed when you registered it with ACM.
13.  Under Job type, choose *Your job will complete after deploying to the selected devices/groups (snapshot)*.
14.  Under IAM role for OTA update job, choose your OTA service role. 
15.  In the ID box, type an alphanumeric ID for your job.
16.  Choose *Create*. 


The job appears with a status of IN PROGRESS in the AWS IoT console window

Note: The AWS IoT console does not update the state of jobs automatically. Refresh your browser to see updates. 
 
Connect your serial UART terminal to your CC3220SF-LAUNCHXL. You should see output similar to what is shown below that indicates the device is downloading the updated firmware.


Note: This output format may change and not exactly match your results as further releases of the software is produced. 

678 603530 [OTA] [OTA] Queued: 2   Processed: 2   Dropped: 0
      679 603959 [OTA Task] [OTA] Set job doc parameter [ jobId: 89cb1be6_e6fa_4260_99cc_54569d9e87dd ]
      680 603960 [OTA Task] [OTA] Set job doc parameter [ streamname: 353c253f-5a06-43b2-99d6-f6041a266db8 ]
      681 603960 [OTA Task] [OTA] Set job doc parameter [ filepath: /sys/mcuflashimg.bin ]
      682 603960 [OTA Task] [OTA] Set job doc parameter [ filesize: 129132 ]
      683 603961 [OTA Task] [OTA] Set job doc parameter [ fileid: 0 ]
      684 603961 [OTA Task] [OTA] Set job doc parameter [ attr: 0 ]
      685 603961 [OTA Task] [OTA] Set job doc parameter [ certfile: tisigner.crt.der ]
      686 603963 [OTA Task] [OTA] Set job doc parameter [ sig-sha1-rsa: bf/ET94tx3PULTzI4VNFxdSDapeK65dn ]
      687 603963 [OTA Task] [OTA] Job was accepted. Attempting to start transfer.
      688 603963 [OTA Task] Sending command to MQTT task.
      689 603963 [MQTT] Received message 80000 from queue.
      690 604065 [MQTT] MQTT Subscribe was accepted. Subscribed.
      691 604065 [MQTT] Notifying task.
      692 604066 [OTA Task] Command sent to MQTT task passed.
      693 604066 [OTA Task] [OTA] Subscribed to topic: $aws/things/TI_CCS3200SF/streams/353c253f-5a06-43b2-99d6-f6041a266db8
      
      694 604530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      695 605530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      696 606530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      697 607530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      698 608530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      699 609530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      700 610530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      701 611530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      702 611776 [OTA Task] [OTA] file token: 74594452
      703 612530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      704 613530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      705 614530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      706 615530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      707 616530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      708 617530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      709 618530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      710 619530 [OTA] [OTA] Queued: 1   Processed: 0   Dropped: 0
      711 620047 [OTA Task] [OTA] file ready for access.
      712 620047 [OTA Task] [OTA] Returned buffer to MQTT Client.
      713 620048 [OTA Task] Sending command to MQTT task.
      714 620048 [MQTT] Received message 90000 from queue.
      715 620048 [MQTT] Notifying task.
      716 620049 [OTA Task] Command sent to MQTT task passed.
      717 620049 [OTA Task] [OTA] Published file request to $aws/bin/things/TI_CCS3200SF/streams/353c253f-5a06-43b2-99d6-f6041a266db8718 620530 [OTA] [OTA] Queued: 3   Processed: 3   Dropped: 0
      719 620653 [OTA Task] [OTA] Received file block 0, size 1024
      720 620667 [OTA Task] [OTA] Remaining: 126
      721 620668 [OTA Task] [OTA] Returned buffer to MQTT Client.
      722 620669 [OTA Task] [OTA] Received file block 1, size 1024
      723 620697 [OTA Task] [OTA] Remaining: 125
      ...

After the device downloads the updated firmware, it restarts and then installs the firmware. You can see what's happening in the UART terminal: 

1285 756833 [OTA Task] [OTA] Resetting MCU to activate new image.
      0 0 [Tmr Svc] Simple Link task created
      
      Device came up in Station mode
      
      1 89 [Tmr Svc] Starting key provisioning...
      2 89 [Tmr Svc] Write root certificate...
      3 190 [Tmr Svc] Write device private key...
      4 290 [Tmr Svc] Write device certificate...
      SL Disconnect...
      
      5 387 [Tmr Svc] Key provisioning done...
      Device came up in Station mode
      
      [WLAN EVENT] STA Connected to the AP: Guest , BSSID: 44:48:c1:ba:b2:c3
      
      [NETAPP EVENT] IP acquired by the device
      
      
      Device has connected to Guest
      
      Device IP Address is 192.168.14.217 
      
      
      6 2440 [OTA] OTA demo version 0.9.3
      7 2440 [OTA] Creating MQTT Client...
      8 2440 [OTA] Connecting to broker...
      9 2440 [OTA] Sending command to MQTT task.
      10 2440 [MQTT] Received message 10000 from queue.
      ...



**WARNING**: Cleaning Up

Once the exercise is complete and the configuration and operation reviewed and understood, the reader is encouraged to remove the configuration from AWS IoT including the certificates, the policy and the “Thing”. Removing these configuration items mitigates unauthorized use of the AWS IoT service should the CC3220SF-LAUNCHXL board be unknowingly removed or borrowed.
