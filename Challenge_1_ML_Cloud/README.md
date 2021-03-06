# Challenge 1: ML in the Cloud

## Summary

Now that you have a face-detection model on a SageMaker endpoint, we can leverage that endpoint to build out an ML/IoT pipeline for our app that does all the inference in the cloud. By running inference in the cloud, we're able to use accurate models at a rapid pace by leveraging powerful computational instances like C5.

In this challenge, you will configure an IoT device to act as a smart camera, which will send images to the endpoint for face-detection. You'll build out the rest of the pipeline necessary to create a dashboard for tracking crowd emotions in real-time, including AWS Lambda, Amazon DynamoDB, Amazon Rekognition, and Amazon CloudWatch.

## Instructions

## Configure GreenGrass on IoT Device
In this part of the lab we’re we will walk you through setting up and entire AWS Greengrass installation including building a simple Greengrass aware client to communicate with your Greengrass core.

### Step 1 - Setting up the hardware	
There are a few things the device is going to need set up at an operating system level to support Greengrass.
We need to add a user and group for Greengrass to use. SSH into your device and run the following commands:

```
sudo adduser --system ggc_user
sudo addgroup --system ggc_group
```

Next we want to edit one of our boot scripts to make sure hardlink/softlink protection is enabled. Edit the following file:

```
sudo nano /etc/sysctl.d/99-sysctl.conf
```

This file is probably empty and you can just add these two lines.

```
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
```

If the above configuration file as correct you can proceed, if you had to add those two lines please reboot your Device and then keep following the steps. To reboot your Device use:

```
sudo reboot now
```

After a few minutes, connect to your instance by using SSH as above. Then, run the following command to confirm the change.

```
sudo sysctl -a | grep fs.protected

```
You should see that hardlinks and softlinks are set to 1.

That completes Greengrass prep. Later in the lab we need to create lambda that will run locally on the Greengrass. That lambda needs some libraries. We will install those libraries on the device as below. On SSH window type commands below

```
pip install jmespath

pip install python-dateutil

sudo  apt-get install python-opencv

pip install opencv-python

pip install boto3

```


### Step 2 - Setting up the Greengrass Group
Next we’re going to set up the Greengrass group and we’re going to be using the AWS IoT Console to complete these steps. The group is how we control which devices can communicate with core as well as the available Lambdas and logging options.

Log into the AWS IoT Console  (aws.amazon.com) and click on Greengrass, and select “Groups” from the left hand menu. 

Select “Create first Group”.

Select “Easy Group Creation”.

Enter a group name, let’s use “MLandIoT”

 Keep the same core name, “MLandIoT_Core”
 
Select “Create Group and Core”.

The wizard is setting up all the Greengrass dependencies that you need. There are quite a few things being done for you:
•	The Group is created
•	The Core thing is created, this is the device that is your Greengrass group
•	Certificates for your core thing are created for you
•	Security policies and logging defaults are also created

On the completion screen you will need to download the certificates and keys for your device! make sure you do this before clicking finish. You will need:
•	The certificate.pem file
•	The private.key file
When you download the certificates, they will have names such as: xxxxxxxxx-certificate.pem.crt and xxxxxxx-private.pem.key
Rename xxxxxxx-certificate.pem.crt to MLandIoT_core.pem.crt
Rename xxxxxxx-private.pem.key to MLandIoT_core.pem.key

It is recommended that you create a working directory to keep all of your files in one place. 
Next, let’s download the Greengrass software for our device:
Select the x86_64 Ubuntu 14.04 - 16.04 software package and click the download button.
 
Download Greengrass Core software software

You can now click the Finish button. You will see the following un-deployed group ready for the next steps.

 
### Step 3 - Set up your Group Role

You need a role attached to your Greengrass group that gives additional permissions to the core to be able to directly access various AWS services.

Navigate to aws.amazon.com and open the Identity and Access Management console (IAM)
 
Click on roles and Create new role

Select AWS Greengrass Role as the Service Role Type – you will have to scroll down to find it in the list.

Select AWSGreengrassResourceAccessRolePolicy, CloudWatchLogsFullAccess, AmazonS3ReadOnlyAccess, AWSGreengrassFullAccess and AmazonSagemakerFullAccess policies and click Next

Enter a name for this role, let’s call it MLandIoTCoreRole
 
Click Create Role

Next we’re going to attach this role to our new Greengrass group.

Open the AWS IoT console

Click on Greengrass > Groups and select the MLandIoT group

Click on Settings, you can see we have no Group Role

Click on Add Role

 

Pick our service role, MLandIoTCoreRole and click Save
 
### Step 4 - Set up logging

We can set up logging so that all logs on the Greengrass core are sent to CloudWatch logs. This includes the logs from the operation of the core as well as the logs from our Lambda functions which are extremely useful to have.


Under "Setting" select CloudWatch logs configuration click Edit

Click on Add another log type

 
Select User Lambdas and Greengrass system

Click Save

 
Keep the log settings as Informational

 
Click Save

### Step 5 - Set up the core endpoints

To be able to connect to Greengrass devices need to have an endpoint. For our workshop we need to define a known endpoint to use in test applications.

While on the settings page for your Group look for the Core connectivity information section

Change this to Manually manage connection information
 
Then click View cores for specific endpoint information

Click on MLandIoT_Core

Click on Connectivity
 
Click Edit

Enter your endpoint information, for this workshop enter your Device IP address 

Enter 8883 for the port.

Click Update

You should see something similar with your Device IP address



### Step 6 - Installing Greengrass on the device

In this step we’re going to take our software package and certificates and get them configured on our Upsquared device 

We will now copy the following to your Upsquared Device :

MLandIoT_core.pem.crt and MLandIoT_core.pem.key file to your home folder which is: /home/upsquared

Copy the Greengrass software (greengrass-ubuntu-x86-64-1.6.0.tar) file to /home/upsquared/

on your local machine, open terminal, change directory to where certs folder is. You can use SCP as below to copy certs and software to the upsquared device

replace your_device_ip to the IP address of the device. You can get IP address of the device by linux command "ifconfig".

```
scp MLandIoT_core.cert.pem  upsquared@your_device_ip:/home/upsquared/
scp MLandIoT_core.private.key  upsquared@your_device_ip:/home/upsquared/

scp greengrass-ubuntu-x86-64-1.6.0.tar upsquared@your_device_ip:/home/upsquared
```

When asked for a password use “upsquared”, 

Windows users can use Filezilla to transfer certs and installation software.

On the Device terminal extract the tar.gz file to the root of your device.

if name of the software is greengrass-ubuntu-x86-64-1.6.0.tar (notice extension is .tar) then

```
sudo tar -xvf ./greengrass-ubuntu-x86-64-1.6.0.tar -C /
```

if name of the software is greengrass-ubuntu-x86-64-1.6.0.tar.gz (notice extension is .gz) then

```
sudo tar -zxvf greengrass-ubuntu-x86-64-1.6.0.tar.gz -C /
```

Copy your certificate and private key to the greengrass certificate folder.

```
sudo cp /home/upsquared/MLandIoT_core.cert.pem /greengrass/certs
sudo cp /home/upsquared/MLandIoT_core.private.key /greengrass/certs
```

Next we need to edit our configuration file

```
sudo nano /greengrass/config/config.json
```

Your configuration file needs to look like the following, notice we specify the location of our Core’s certificates, the Root CA file to validate the identity of AWS IoT and our endpoint information:

```
{
   "coreThing": {
       "caPath": "root-ca.pem",
       "certPath": "MLandIoT_core.cert.pem",
       "keyPath": "MLandIoT_core.private.key",
       "thingArn": "MLandIoT_Core ARN",
       "iotHost": "HOST_PREFIX_HERE.iot.AWS_REGION_HERE.amazonaws.com",
       "ggHost": "greengrass.iot.AWS_REGION_HERE.amazonaws.com",
       "keepAlive": 600
   },
   "runtime": {
       "cgroup": {
           "useSystemd": "yes"
       }
   },
"manageRespawn": false
}
```

Note: You will need to fill in the uppercase info with your information as outlined below:

The thingArn is obtained from your account:
Click on Core while on your Greengrass group

Click on the MLandIoT_Core

The ARN is listed on that page 

You will also need your IoT endpoint which is on the settings page of your AWS IoT console page.

Your final config file should look something like this:
```
{
   "coreThing": {
       "caPath": "root-ca.pem",
       "certPath": "MLandIoT_core.cert.pem",
       "keyPath": "MLandIoT_core.private.key",
       "thingArn": "arn:aws:iot:us-xxxx-x:xxxxxxxxx:thing/MLandIoT_Core",
       "iotHost": "xxxxxxxx.iot.us-xxxx-x.amazonaws.com",
       "ggHost": "greengrass.iot.us-xxxx-x.amazonaws.com",
       "keepAlive": 600
   },
   "runtime": {
       "cgroup": {
           "useSystemd": "yes"
       }
  },
"manageRespawn": false
}
```

Lastly, we need to get a copy of the Root CA file for server identity verification.

change directory to /greengrass/certs. Enter this on your device to obtain a copy of the certificate:

```
sudo wget -O root-ca.pem  https://www.symantec.com/content/en/us/enterprise/verisign/roots/VeriSign-Class%203-Public-Primary-Certification-Authority-G5.pem
```

### Step 7 - Starting the Greengrass core

We are now ready to run the Greengrass Core! Once we see that it is running we will then deploy the group configuration from the console.
To make the next steps easier I suggest running as root.
```
sudo su
cd /greengrass/ggc/core
./greengrassd start
```

To make sure that your core is running and listening for messages on port 8883 enter:
```
netstat -peanut | grep -i 8883 
 ```

The log files are located in /greengrass/ggc/var/log/system/*

You want to tail the runtime log to see if the certificates and endpoints have been accepted.

```
tail -f /greengrass/ggc/var/log/system/runtime.log
```

You should see messages similar to the following:

```
[INFO]-Deployment agent connected to cloud
[INFO]-Subscribed to topic $aws/things/GGC_Thing-gda/shadow/update/delta
[INFO]-Subscribed to topic $aws/things/GGC_Thing-gda/shadow/get/accepted
```

###Step 8 - Deploying the Greengrass Group

Now that you have the core running, let’s deploy our group to the device.

On your group management page, click on Deployments

Click on Actions and Deploy

Select – Automatically detect Core endpoints (recommended)

Click next

You should see deployment successfully completed

If you don’t get a successful deployment then your core might not be operating correctly. Please verify Step 7.

Once it is deployed you can move onto the next Greengrass lab. So far you have a functioning core but it doesn’t do much. In the next section we’ll set up a full group and connect a test device.

Congrats on getting Greengrass running!


### Deploy Lambda Code to IoT Device


### Lambda setup:
Go to Lambda Console

Create function from template

Select Greegrass python template, Next

Give name greengrassSagemakerInference

Under role, choose lambda_basic_execution

Save and Create function

There will be sample code in this lambda

Open greengrassSagemakerInference.py, check line #44, endpoint_name = 'xxxxx', replace xxxx with actual SagemakerEndpoint 
you created during Sagemaker prechallenge lab. Save the file.

Remove all code from lambda greengrassHelloWorld. Copy all code from greengrassSagemakerInference.py with updated endpoint and paste it inside lambda greengrassHelloworld.py

Deploy Lambda (GG Group Settings with the new Lambda and Resources)

Lets publish this lambda function;

Select Actions dropdownbox

Click "Publish new version"
keep Publish box empty, Click on "Publish".

This will create version 1 for the new function. Now we can go back to Greengrass console to complete our group settings before the deployment.

3.	Add Lambda to the Group
Groups -> MLandIoT -> Lambdas -> Add Lambda -> Use exiting Lambda -> greengrassSagemakerInference -> Version 1 ->Finish

Under Groups -> Lambdas, select greengrassSagemakerInference. Click Edit button.

Edit lambda memory limit and timeout like below

    - Memory limit: 96 mb
    - Timeout: 10 sec
    - Check "Make this function long-lived and keep it running indefinitely" 
 
 Click "Update" to save the changes.
	    	   
4.	Add local dev resources for the camera access
Under Groups -> MLandIoT ->Resources
Select Add Resource

*	Name this resource - webcam

*	Local resource type – Device

*	Device path - /dev/video0

*	Specify the OS group used to access this resource: Select "Automatically add OS group"

*	Select "Read and write access"

Save

Under Groups -> Lambdas, select greengrassSagemakerInference

5.	Attach the resources to Lambda

*	Add resources – select webcam

6.	Add Subscriptions
Go to Subscriptions menu located on left side, then add subscription, select source as lambda – greengrassSagemakerInference, target as IoT Cloud and Topic as “ModelInference” 
	 

7.	Deploy Greengrass group:

Groups -> YourGroup -> Actions -> Deploy

Evaluate the Model @Cloud

1.	Subscribe to "ModelInference" topic via AWS IoT Console / Test interface on console.

2.	You should be able to see prediction results flooding in


Congratulations Greengrass setup and configuration is done!!!


## Create Rekognition Lambda

Now that we've configured our IoT device to send images to our SageMaker endpoint, we've completed the first two stages of our pipeline. The last stages include processing our face crops through Rekognition to extract emotion scores, storing scores in a DynamoDB table, and pushing scores to Cloudwatch so we can build a dashboard to track emotion metrics.

We're going to use AWS Lambda to create a script that does all three of these things every time a face crop is pushed to S3. Navigate to the Lambda console and under the "Function" Dashboard select "Create function" as you did before. This time we'll be authoring a function from scratch:

![Alt text](../screenshots/create_lambda_0.png)

Next:
* Under Name: enter your rekognition lambda name
* Under runtime: enter "Python 2.7"
* Under role: Select "Create new role from template(s)"
* Under role name: Enter your rekogition lambda role name
* Under Policy templates: Add "S3 object read-only permissions"

Then click "Create function."

![Alt text](../screenshots/create_lambda_1.png)

You should now be on the lambda function screen. Before we start editing the lambda, we need to assign additional permissions to the lambda role. As you can tell from the services listed on the right, we currently only have access to "Amazon CloudWatch Logs" and "Amazon S3".

![Alt text](../screenshots/edit_lambda_0.png)

We need to add permissions for Rekognition and DynamoDB. Navigate to the IAM dashboard by searching for "IAM" in the "Services" drop-down. Once there, click "Roles" on the left side-bar and type in your rekognition lambda role name.

![Alt text](../screenshots/add_permissions_0.png)

Select your role, and you should see two policies already created from the template.

![Alt text](../screenshots/add_permissions_1.png)

Select "Attach Policy". First, search for "DynamDB", and select "AmazonDynamoDBFullAccess". 

![Alt text](../screenshots/add_permissions_2.png)

Next, search for "Rekognition" and select "AmazonRekognitionFullAccess".

![Alt text](../screenshots/add_permissions_3.png)

Then, click "Attach policy", and you should now see these policies attached to your role.

![Alt text](../screenshots/add_permissions_4.png)

Back on the lambda function page, we can now see additional resources available to us on the right. 

![Alt text](../screenshots/edit_lambda_1.png)

We know that we want this lambda script to run everytime a face crop is uploaded to S3, so let's add an event trigger to this Lambda. On the left, you'll see a list of triggers. Select "S3". At the bottom of the page, a configuration menu will open up:
* Under Bucket: Select the bucket you created to store faces (this will be different from mine).
* Under Event type: Select "PUT". We want the script to trigger when a PutObject call is made.
* Under Prefix: Enter "faces". We want the script to only trigger on items uploaded to the faces prefix.

![Alt text](../screenshots/edit_lambda_2.png)

Then click "Add". Next, select the center box with your rekognition lambda's name. The menu at the bottom of the page will now let you manually enter the function code.

![Alt text](../screenshots/edit_lambda_3.png)

Next, you're going to copy and paste code we've provided you in this repo into the text editor in the lambda dashboard. You can find it under Challenge_2, "rekognize-emotions.py", but we've included it here as well for your convenience:

```python
from __future__ import print_function

import boto3
import urllib
import datetime

print('Loading function')

rekognition = boto3.client('rekognition')
cloudwatch = boto3.client('cloudwatch')


# --------------- Helper Function to call CloudWatch APIs ------------------

def push_to_cloudwatch(name, value):
    try:
        response = cloudwatch.put_metric_data(
            Namespace='string',
            MetricData=[
                {
                    'MetricName': name,
                    'Value': value,
                    'Unit': 'Percent'
                },
            ]
        )
        print("Metric pushed: {}".format(response))
    except Exception as e:
        print("Unable to push to cloudwatch\n e: {}".format(e))
        return True

# --------------- Helper Functions to call Rekognition APIs ------------------

def detect_faces(bucket, key):
    print("Key: {}".format(key))
    response = rekognition.detect_faces(Image={"S3Object":
                                               {"Bucket": bucket,
                                                "Name": key}},
                                        Attributes=['ALL'])

    if not response['FaceDetails']:
        print ("No Face Details Found!")
        return response

    push = False
    dynamo_obj = {}
    dynamo_obj['s3key'] = key

    for index, item in enumerate(response['FaceDetails'][0]['Emotions']):
        print("Item: {}".format(item))
        if int(item['Confidence']) > 10:
            push = True
            dynamo_obj[item['Type']] = str(round(item["Confidence"], 2))
            push_to_cloudwatch(item['Type'], round(item["Confidence"], 2))

    if push:  # Push only if at least on emotion was found
        table = boto3.resource('dynamodb').Table('rekognize-faces')
        table.put_item(Item=dynamo_obj)

    return response

# --------------- Main handler ------------------


def lambda_handler(event, context):
    '''Demonstrates S3 trigger that uses
    Rekognition APIs to detect faces, labels and index faces in S3 Object.
    '''

    # Get the object from the event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.unquote_plus(event['Records'][0]['s3']['object']['key'].encode('utf8'))
    try:
        # Calls rekognition DetectFaces API to detect faces in S3 object
        response = detect_faces(bucket, key)

        return response
    except Exception as e:
        print("Error processing object {} from bucket {}. ".format(key, bucket) +
              "Make sure your object and bucket exist and your bucket is in the same region as this function.")
        raise e
```

Plese take a look at what this script does. The Function `lambda_handler` handles the lambda script when it's triggered by an event, in this case the "PutObject" to your S3 bucket under the prefix "faces". The handler then calls the `detect_faces`, which does the following:

* Makes a `detect_faces` API call to Rekognition, handling an empty response
* Checks if any emotion scores are greater than 10
* If so, pushes emotion type and confidence score to CloudWatch
* If at least one emotion in the response is detected or significant, the record is stored in a DynamoDB table.

Once you've copy and pasted the code, we're almost ready to Save the function so it can start triggering.

![Alt text](../screenshots/edit_lambda_4.png)

Before we do that, we need to create the DynamoDB table to store detected emotions and emotion scores.

## Create DynamoDB Table

Navigate to the DynamoDB dashboard by seraching "DynamDB" in the "Services" drop-down tab. Click "Create table" on the dashboard page (it may look different if you've created a table before).

![Alt text](../screenshots/dynamodb_0.png)

Next:
* Under Table name: Enter the table specified in the lambda function, "rekognize-faces"
* Under primary key: Enter "s3key".

![Alt text](../screenshots/dynamodb_0.png)

Then click create. Once created, go back to your lambda function and make sure to click "Save" at the top right. You lambda function is now active and should begin triggering upon face crop uploads.

## Emotion-tracking Dashboard using Cloudwatch

Now that we've created the lambda function for processing cropped faces and a DynamoDB table to record results, we're going to build the dashboard that is the center-piece of our application: real-time emotion tracking. At this point, you should have your IoT device running and collecting face crops if it hasn't been already.

Navigate to CloudWatch by searching for "CloudWatch" under the "Services" tab. Once at the dashboard, select "Dashboards" from the left side-bar.

![Alt text](../screenshots/cloudwatch_0.png)

Select "Create Dashboard", then enter your dashboard name.

![Alt text](../screenshots/cloudwatch_1.png)

Then select "Line" widget and click "Configure".

![Alt text](../screenshots/cloudwatch_2.png)

You'll then be able to configure your line widget. 

![Alt text](../screenshots/cloudwatch_3.png)

* Click on "Untitled graph" at the top left to name your graph. 
* In the "Metrics" menu, you should see a section that says "Custom Namespaces". 
* Click on "string" (if this is not there, your face crops haven't made it through the pipeline yet. Wait a few minutes).
* Click on "Metrics with no dimensions"
* Select all available metrics (there should be 7. These populate as the emotions are detected. If they're not all there, make some funny faces at your camera to get a range of detections.)

Then, click the tab "Graphed metrics"

![Alt text](../screenshots/cloudwatch_4.png)

Here, you can change how your metrics are displayed. By default, point metrics shown are averages of 5 min. periods. You may want to change the granularity to a smaller period to actually start seeing line graphs being drawn.

In addition, make sure to turn on "Auto refresh" and set the interval to 10 seconds.

![Alt text](../screenshots/cloudwatch_5.png)

Finally, click "Create Widget" and you'll have successfully created the emotion tracking dashboard for your application! Now wait and see the results come in as your IoT device sends frames through your pipeline to detect emotions over time.

**To-do: CloudWatch Dashboard image with extensive data collected**












