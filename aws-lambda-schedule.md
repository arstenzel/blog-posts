## Introduction 

The purpose of this blog post is to illustrate how one can use lambda functions 
to securely query an environment and send the output to slack. 

I realize that most readers will not be users of Ravello. But the overall process 
can be used for any type of system query. 

For example, just remove the Ravello portions in the `handler` function (below) 
and substitute your own code to query a system of your choice

Ex.
```python
def handler(event, context):
   post_to_slack(some_cool_function_to_query_some_other_system_you_choose())
```
 
In addition to this post, all the code described herein can be found on GitHub 
at [https://github.com/Solinea/lambda-ravello][4]. The code works and currently 
runs in production.

Between the GitHub repository contents and the descriptions below, you should 
get a fairly complete picture of how useful lambdas can be for everyday chores. 

## Prerequisites 

This assumes you have a working AWS ID, have installed the AWS Commandline Tools and Python 2.7.

## Background 
We have a test environment called Ravello in which we have several VMs. The VMs 
are supposed to run only during normal business hours.  We take them down at the 
end of each day to avoid additional costs.

The problem was detecting when VMs were left running.  There was no notification 
mechanism.  We simply did not know when a VM was accidently left active for hours 
or even days. While this seems trivial, the costs mount up and can be substantial.

What would a notification process look like?  We use Slack for all our internal 
communications.  Posting a message there would be ideal. Naturally, all of this 
would have to be secure. We would not want to expose user IDs or passwords 
anywhere.

Therefore, we have 3 requirements: 
1. That we are notified of any running VMs outside business hours 
1. That these notifications go to Slack.
1. That communication be secure 

## The Solution

This is where AWS lambda functions come in handy.  We can use a lambda function 
to periodically query Ravello (#1 below), report the output on Slack (#2), and 
show it on one of our channels (#3). 
 
![Ravello Flow](/images/Ravello%20Flow.jpg)

## Creating The AWS Lambda Role
Before we can upload a new lambda function to AWS, we need to create an IAM role 
to support it. This is called an IAM Execution Role. Here are the steps
1. Sign in to the IAM console at https://console.aws.amazon.com/iam/
1. Follow the steps in ["Creating a Role for an AWS Service"][3] in the IAM User Guide to create an IAM role (execution role). As you follow the steps to create a role, note the following:
    1. In Role Name, use a name that is unique within your AWS account (for example, `lambda_ravello`).
    1. In Select Role Type, choose AWS Service Roles, and then choose AWS Lambda. This grants the AWS Lambda service permissions to assume the role.
    1. In Attach Policy, choose `AWSLambdaBasicExecutionRole`.


## Creating The Slack Incoming Web Hook
1. On Slack, go to the channel where you would like your lambda messages to be posted 
1. Click on the "Conversation Settings" icon (it looks like a little cog)
1. Select "Add an app or integration". This will take you to a web page.
1. Search for and select "Incoming Webhooks" 
1. Click on the green "Add Configuration" button
1. Pick the channel you want and then click the green "Add Incoming Webhooks Integration"
1. COPY AND SAVE the Webhook URL displayed
1. Select the green "Save Setting" button at the bottom
1. That's it. You now have an incoming webhook for that channel.

## Source Code
For the lambda function itself, all you need is a directory and a couple source files.  

Make a directory somewhere on your local machine called “audit”.  In it, 
create 2 files, `audit.py` and `lambda.json`

```
mkdir audit
cd audit
touch audit.py lambda.json
```

Set the content of `lambda.jason` to the following.  It is used primarily by the `lambda-uploader` utility described below.

```json
{
  "name": "audit",
  "description": "Count of all running VMs",
  "region": "us-east-1",
  "handler": "audit.handler",
  "role": "arn:aws:iam::999999999999:role/lambda_ravello",
  "requirements": [],
  "ignore": [
    "/*.pyc"
  ],
  "timeout": 30,
  "memory": 128
}
```
Notes: 
  1. Make sure you set the `region` to match your own
  1. Substitute your account number for “999999999999”


Set the contents of `audit.py` to the following. This is the actual lambda function used in AWS. 

```python

"""

Description:
 A simple lambda function which will run a quick audit of Ravello VMs.

"""
from __future__ import print_function
import logging
import boto3
import pycurl
from io import BytesIO
import json

from base64 import b64decode
from urllib2 import Request, urlopen, URLError, HTTPError

SLACK_CHANNEL = '#play'  

logger = logging.getLogger()
logger.setLevel(logging.INFO)

ENCRYPTED_RAVELLO_USER = 
DECRYPTED_RAVELLO_USER = boto3.client('kms').decrypt(CiphertextBlob=b64decode(ENCRYPTED_RAVELLO_USER))['Plaintext']
ENCRYPTED_RAVELLO_PASSWORD = 
DECRYPTED_RAVELLO_PASSWORD = boto3.client('kms').decrypt(CiphertextBlob=b64decode(ENCRYPTED_RAVELLO_PASSWORD))['Plaintext']

ENCRYPTED_SLACK_HOOK_URL = 
DECRYPTED_SLACK_HOOK_URL = boto3.client('kms').decrypt(CiphertextBlob=b64decode(ENCRYPTED_SLACK_HOOK_URL))['Plaintext']
SLACK_HOOK_URL = "https://%s" % DECRYPTED_SLACK_HOOK_URL


def post_to_slack(msg):
    slack_message = {
        'channel': SLACK_CHANNEL,
        'text': msg
    }
    req = Request(SLACK_HOOK_URL, json.dumps(slack_message))
    try:
        response = urlopen(req)
        response.read()
        logger.info("Message posted to %s", slack_message['channel'])
    except HTTPError as e:
        logger.error("Request failed: %d %s", e.code, e.reason)
    except URLError as e:
        logger.error("Server connection failed: %s", e.reason)


def handler(event, context):

    total_active = 0
    buf = BytesIO()
    c = pycurl.Curl()
    c.setopt(c.URL, 'https://cloud.ravellosystems.com/api/v1/applications')
    c.setopt(c.WRITEFUNCTION, buf.write)
    c.setopt(c.HTTPHEADER, ['Accept: application/json'])
    c.setopt(c.USERPWD, "%s:%s" % (DECRYPTED_RAVELLO_USER,DECRYPTED_RAVELLO_PASSWORD))
    c.perform()
    j = json.loads(buf.getvalue())
    for item in j:
        if "deployment" in item:
            active = item["deployment"]["totalActiveVms"]
            total_active += active
            if active:
                msg = "Lab: %s, Owner: %s, VMs: %d" % (item["name"], item["owner"], active)
                post_to_slack(msg)
    post_to_slack("Ravello total active VMs: %d"%total_active)

```


Simple, right?  Well, not completely.  There are a couple security issues we have to address.  
Notice there are 3 variables which aren't initialized in the above source. They 
are `ENCRYPTED_RAVELLO_USER`, `ENCRYPTED_RAVELLO_PASSWORD` and `ENCRYPTED_SLACK_HOOK_URL`.  
To supply these values, we have to do some work on AWS first.

## Creating The AWS Encryption Key

To encrypt our secrets, we first need to create a key for them on AWS. To do that 
1. Sign in to the IAM console at https://console.aws.amazon.com/iam/
1. Select Encryption Keys
1. Press the Create key button
1. Give the new key an alias (e.g. `lambda_key`)
1. On the Key Administrator and Key User panels, select the Lambda Role `lambda_ravello` we created (above). 
1. Select `Next` until completed
1. You should now have a new entry called `lambda_key` in your list of keys


## Using The AWS Encryption Key
What we're going to do now is encrypt our Ravello user id/password and as well as 
our Slack webhook url.  This will allow us to finally add values to the uninitialized 
variables in our `audit.py` file above.


1. For each of the entities Ravello User ID, Ravello Password, and Slack Webhook URL, do the following
    1. Go to your local command line and encrypt your strings using the
    newly-created key like this:
       ```
       aws kms encrypt --key-id alias/lambda_key --plaintext "<thing to be encrypted>"
       ```
    1. The output should look something like this:
       ```bash
       {
       "KeyId": "arn:aws:kms:us-east-1:9999999999:key/aac247d3-388a-47cd-b511-89eace8ec389",
       "CiphertextBlob": "AQICAHhNSZNrOg1IZ0wkoZle+c+FHFCaI32SkfaXfYhnbR5jdQFhKPWC6QOVeNpoqmhAMkcbAAAAZDBiBgkqhkiG9w0BBwagVTBTAgEAME4GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMBp0IGcIiNmfPa3QXAgEQgCGpyLSoVewQcynE6ZVZRuWjb/KjGb5J0K5n6/E4yiJ+MkI="
       }
       ```
    1. The resulting output in `CiphertextBlob` will be used for the content of the uninitialized variables in our `audit.py` source code above. For example if the `plaintext` content string is the Ravello User Id, then the `CiphertextBlob` content will be used for `ENCRYPTED_RAVELLO_USER`.



## Upload Utility

Now we're almost ready to upload our new lambda function. To make lambda creation 
and uploading easier, let's install a python utility called [`lambda-uploader`][1]. 
We will use this to stage our lambda function on AWS.

```bash
pip install lambda-uploader
```

## Upload The New Lambda Function 
To upload our lambda function, go to the parent directory of the `audit` directory. Then issue the command 
```bash
$ lambda-uploader --publish  ./audit --config ./audit/lambda.json
```

If successful, the output should look like this
```bash
λ Building Package
λ Uploading Package
λ Fin
```

## Associating The Lambda Function With Our Encryption Key
Even though we now have successfully uploaded our lambda function, we still have 
more to do.  First, we need to tell the function which key to use for deciphering 
the encrypted strings in our `audit.py` file. Do the following

1. Sign in to the Lambda console at https://console.aws.amazon.com/lambda/
1. Click on our function `audit`
1. Select "Configuration" from the tabs listed on the page
1. Select the "Advanced" fold-down menu
1. Scroll to the bottom of the page until you see the "KMS Key" field
1. Select our key `lambda_key` from the fold-down list of keys.
1. Press the "Save" button at the top of the page.
1. At this point, if you press "Test" or "Save And Test", you should see output go to your Slack channel.


## Setting The Lambda Function To Execute On A Timed Basis
Now we need to define *when* our function should be executing. 

1. Sign in to the Lambda console at https://console.aws.amazon.com/lambda/
1. Click on our function `audit`
1. Select "Triggers" from the tabs listed
1. Click on "Add Trigger"
1. Click in the dashed icon and you will get a list of options
1. Select "Cloudwatch Events"
1. Select "Create A New Role"
1. Set the new role's name (e.g. `ravello`)
1. Set the "Schedule Expression" in the bottom field (e.g.`cron(0 17 ? * MON-FRI *)`). This tells the trigger when to execute the lambda function.
1. Click "Submit"
1. You're Done! Now all you have to do is wait until the timer trips and you should see output on your Slack channel!



[1]: https://github.com/rackerlabs/lambda-uploader
[2]: https://docs.aws.amazon.com/lambda/latest/dg/intro-permission-model.html#lambda-intro-execution-role
[3]: http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html
[4]: https://github.com/Solinea/lambda-ravello
