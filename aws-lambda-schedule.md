We have a test environment called Ravello in which we have several VMs. The VMs are supposed to run only during normal business hours.  We take them down at the end of each day to avoid additional costs.

The problem was detecting when VMs were left running.  There was no notification mechanism.  We simply did not know when a VM was accidently left active for hours or even days. While this seems trivial, the costs mount up and can be substantial.

What would a notification process look like?  We use Slack for all our internal communications.  Posting a message there would be ideal.

Therefore, we have 2 requirements: 
1. That we are notified of any running VMs outside business hours 
1. That these notifications go to Slack.

This is where AWS lambda functions come in handy.  We can use a lambda function to periodically query Ravello (#1 below) and report the output on Slack (#2) to one of our channels (#3). 
 
![Ravello Flow](/images/Ravello%20Flow.jpg)

