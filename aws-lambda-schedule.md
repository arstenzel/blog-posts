## Prerequisites 

This assumes you have a working AWS ID and have installed the AWS Commandline Tools.


## Background 
We have a test environment called Ravello in which we have several VMs. The VMs are supposed to run only during normal business hours.  We take them down at the end of each day to avoid additional costs.

The problem was detecting when VMs were left running.  There was no notification mechanism.  We simply did not know when a VM was accidently left active for hours or even days. While this seems trivial, the costs mount up and can be substantial.

What would a notification process look like?  We use Slack for all our internal communications.  Posting a message there would be ideal.

Therefore, we have 2 requirements: 
1. That we are notified of any running VMs outside business hours 
1. That these notifications go to Slack.

This is where AWS lambda functions come in handy.  We can use a lambda function to periodically query Ravello (#1 below) and report the output on Slack (#2) to one of our channels (#3). 
 
![Ravello Flow](/images/Ravello%20Flow.jpg)

## AWS Lambda Role
Before we can upload a new lambda function to AWS, we need to create an IAM role to support it. This is called an IAM Execution Role. Here are the steps
1. Sign in to the IAM console at https://console.aws.amazon.com/iam/
1. Follow the steps in ["Creating a Role for an AWS Service"][3] in the IAM User Guide to create an IAM role (execution role). As you follow the steps to create a role, note the following:
    1. In Role Name, use a name that is unique within your AWS account (for example, `lambda_ravello`).
    1. In Select Role Type, choose AWS Service Roles, and then choose AWS Lambda. This grants the AWS Lambda service permissions to assume the role.
    1. In Attach Policy, choose `AWSLambdaBasicExecutionRole`.

## AWS Encryption Keys

To encrypt our secrets, we first need to create a key for them on AWS.  We do that by 
 1. Going to the main AWS console and selecting IAM->Encryption Keys->Create key
 
 1. Select the Lambda Role we created (above) for both the Key Administrator and Key User

 1. Go to your local command line and encrypt your strings using the
    newly-created key (e.g. `foo`) like this:
       ```
       aws kms encrypt --key-id alias/foo --plaintext "secret"
       ```

 1. The resulting output are the ENCRYPTED_<whatever> values pasted below


## Source Code
For the lambda function itself, all you need is a directory and a couple source files.  

Make a directory somewhere on your local machine called “audit”.   In it, create 2 files, `audit.py` and `lambda.json`


```
mkdir audit
cd audit
touch audit.py lambda.json
```

Set the content of `lambda.jason` to the following:

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


Now, set the contents of `audit.py` to the following. This is the actual lambda function we will upload to AWS. No other source files are needed.

```python

"""

Description:
 A simple lambda function which will run a quick audit Ravello VMs.

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


Simple, right?  Well, not completely.  There are a couple security issues we have to address.  Notice there are 3 variables which aren't initialized in the above source. They are `ENCRYPTED_RAVELLO_USER`, `ENCRYPTED_RAVELLO_PASSWORD` and `ENCRYPTED_SLACK_HOOK_URL`.  To supply these values, we have to do some work on AWS first.




## Upload Utility

To make lambda creation and uploading easier, let's install a python utility called “lambda-uploader”. We will use this to stage our lambda function on AWS.

```bash
pip install lambda-uploader
```




[1]: https://github.com/rackerlabs/lambda-uploader
[2]: https://docs.aws.amazon.com/lambda/latest/dg/intro-permission-model.html#lambda-intro-execution-role
[3]: http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html
