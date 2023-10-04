## Filter and stream logs from Amazon S3 logging buckets into Splunk using AWS Lambda

## Introduction


AWS Customers of all sizes – from growing startups to large enterprises, manage multiple AWS accounts. Following the prescriptive guidance from AWS for multi-account management, customers typically choose to perform centralization of the AWS log sources (CloudTrail logs, VPC Flow logs, Config logs etc.) from multiple AWS accounts within Amazon S3 buckets in a dedicated log archive account.
 
Depending on the number of AWS accounts and the size of workloads, the volume of logs stored in these centralized S3 buckets can be extremely high (multiple TBs/day). In order to ingest the logs from these S3 buckets in Splunk, customers normally use the Splunk Add-on for AWS that is deployed on Splunk Heavy Forwarders to pull the logs from S3. This approach needs dedicated pollers (typically Heavy Forwarders on EC2 instances) to pull the data from S3 and they need to scale horizontally as the data ingestion volume increases to support near-real time ingestion of logs. This approach is not elastic, incurring costs for the structure, even with low log volume, making pull-based ingestion with Heavy Forwarders not the most cost-efficient solution for log ingestion.
 
Consider a use case where you want to save ingest license costs in Splunk by filtering and forwarding only a subset of the data from the S3 logging buckets to Splunk. Example: Ingesting only the rejected traffic within the VPC Flow logs  (where the field “action” == “REJECT”)? The pull-based log ingestion approach does not offer an easy way to achieve that. 

## Prerequisites

The following pre-requisites exist at a minimum:

* AWS account – If you don’t have an AWS account, follow [these steps](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html) to create one.
* Publish VPC Flow logs to Amazon S3 – [Configure VPC Flow logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-s3.html) to be published to an Amazon S3 bucket within your AWS account.

## Splunk HTTP Event Collector (HEC) Configuration

We need to set up Splunk HEC to receive the data before we can configure the AWS services to forward the data to Splunk. 

1. Access Splunk Web, go to Settings and choose Data inputs. 
2. Select HTTP Event Collector and then choose New Token. 
3. Configure the new token and click Submit. Ensure that the sourcetype is “aws:cloudwatchlogs:vpcflow”
4. Once the Token has been created, choose Global Settings, ensure All Tokens have been enabled, and click Save.

## Splunk Configurations

We need to add configurations within the Splunk server under props.conf to ensure that line breaking, time stamp and field extractions are configured correctly. Copy the contents below in props.conf in $SPLUNK_HOME/etc/system/local/. For more information regarding these configurations, refer the Splunk [props.conf documentation](https://docs.splunk.com/Documentation/Splunk/latest/Admin/Propsconf).

[aws:cloudwatchlogs:vpcflow]  
BREAK_ONLY_BEFORE_DATE = false  
CHARSET=UTF-8   
disabled=false  
pulldown_type = true  
SHOULD_LINEMERGE=false  
LINE_BREAKER=\'\,(\s+)  
NO_BINARY_CHECK=true  
KV_MODE = json  
TIME_FORMAT = %s  
TIME_PREFIX = ^(?>\S+\s){10}  
MAX_TIMESTAMP_LOOKAHEAD = 10  #makes sure account_id is not used for timestamp  
#Replace unrequired characters from the VPC Flow logs list with blank values  
SEDCMD-A = s/\'//g  
SEDCMD-B = s/\"|\,//g

#Extraction of fields within VPC Flow log events    
EXTRACT-vpcflowlog=^(?<version>2{1})\s+(?<account_id>[^\s]{7,12})\s+(?<interface_id>[^\s]+)\s+(?<src_ip>[^\s]+)\s+(?<dest_ip>[^\s]+)\s+(?<src_port>[^\s]+)\s+(?<dest_port>[^\s]+)\s+(?<protocol_code>[^\s]+)\s+(?P<packets>[^\s]+)\s+(?<bytes>[^\s]+)\s+(?<start_time>[^\s]+)\s+(?<end_time>[^\s]+)\s+(?<vpcflow_action>[^\s]+)\s+(?<log_status>[^\s]+)

## Create Amazon SQS to queue event notifications 

Whenever a new object (log file) is stored in S3 bucket, an event notification is forwarded to this SQS queue. We need to create this SQS queue and then configure a log centralization S3 bucket to forward event notifications.

1. Access the Amazon SQS console in your AWS Account and choose Create Queue.
2. Select Standard type and choose a Queue name.
3. Within Configurations, increase the Visibility timeout to 5 minutes, and the Message retention period to 14 days. 
4. Enable Encryption for at-rest encryption for your queue.
5. Configure the Access policy as shown below to provide the S3 bucket with permissions to send messages to this SQS queue. Replace the placeholders in <> with the specific values for your environment.

`{
    "Version": "2012-10-17",
    "Id": "Queue1_Policy_UUID",
    "Statement": [
      {
        "Sid": "Queue1_Send",
        "Effect": "Allow",
        "Principal": {
          "Service": "s3.amazonaws.com"
        },
        "Action": "sqs:SendMessage",
        "Resource": "<arn_of_this_SQS>",
        "Condition": {
          "StringEquals": {
            "aws:SourceAccount": "<your_AWS_Account_ID>"
          },
          "ArnLike": {
            "aws:SourceArn": "<arn_of_the_log_centralization_S3_bucket>"
          }
        }
      }
    ]
  }`

6. Enable Dead-letter queue so that any messages that are not processed from this queue will be forwarded to the dead-letter queue for further inspection.

## Forward Amazon S3 Event Notifications to Amazon SQS

Now that the SQS queue has been created, we need to configure the VPC Flow log S3 bucket to forward the event notifications for all Object Create events to the queue.

1. From the Amazon S3 console, access the centralized S3 bucket for VPC Flow logs.
2. Select the Properties tab, scroll down to Event notifications, and choose Create event notifications.
3. Within General configurations, provide an appropriate Event name. Under Event types, select All object create events. Under Destination, choose SQS queue and select the SQS queue that we created in the previous step fro the dropdown. Click on Save changes.

## Create a backsplash Amazon S3 bucket

Let’s create a backsplash S3 bucket to ensure that no filtered data is lost, in case the Lambda function is unable to deliver data to Splunk. The Lambda function will send the filtered logs to this bucket whenever the delivery to Splunk fails. Please follow the steps [here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html) to create an S3 bucket.

## Create an IAM Role for the Lambda function

1. From the AWS IAM Console, access the Policies menu and select Create Policy.
2. Select JSON as the Policy Editor option and paste the policy below into the Policy editor. Replace the placeholders in <> with specific values for your environment. Once done, click Next.

`{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "lambdaPutObject",
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<your_backsplash_s3_name>/*"
        },
        {
            "Sid": "lambdaGetObject",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<your_log_centralization_s3_name>/*"
        },
        {
            "Sid": "lambdaSqsActions",
            "Effect": "Allow",
            "Action": [
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:GetQueueAttributes"
            ],
            "Resource": "<arn_of_the_SQS_Queue>" 
        }
    ]
}`

3. Enter a name and description for the policy, and select Create Policy.
4. From the IAM Console, access Roles and select Create role.
5. Under Use Case, select Lambda and click on Next.
6. On the Add Permissions page, select the AWS managed AWSLambdaBasicExecutionRole policy and the custom policy that we just created prior to creating this role. Choose Next once both the policies are selected.
7. Enter an appropriate role name and then choose Create role.

## Create Lambda Function to filter and push logs to Splunk

1. Access the AWS Lambda console and choose Create function.
2. Under Basic information, enter an appropriate Function name and under Runtime, choose the latest supported runtime for Python.
3. Expand the Change default execution role option, select Use an existing role and select the role that we created in the previous section from the drop-down.
4. Keep all the other settings as default and select Create function.
5. Once the function is created, select the Configuration tab within the function and edit the General configuration. Change the Timeout value to 5 min and click Save.
6. Next, edit the Environment variables and add these 3 key-value pairs. Make sure that you replace the placeholders in <> with the appropriate values based on your environment. Once the environment variables are added, click Save:  
- backup_s3 = name of the backsplash S3 bucket created in the earlier section
- splunk_hec_token = <your_splunk_hec_token>  
- splunk_hec_url = <your_splunk_url>:8088/services/collector/raw


7. Select the Code tab within your Function and update the lambda_function.py with the code from this GitHub repository [here](https://github.com/aws-samples/vpc-flow-log-filtering-and-ingestion-in-splunk-using-aws-lambda/blob/main/lambda_splunk_function.py)
8. Access the Configuration tab within the function and select Triggers.
9. Click Add trigger and select SQS from the dropdown.
10. Under the SQS queue dropdown, select the SQS that we configured to store the S3 object-create event notifications.
11. Select Activate trigger. Keep all the other settings as default and select Add.

## Searching the VPC Flow Logs in Splunk
Once the Lambda function is created and the SQS trigger has been activated, the function immediately starts forwarding the VPC Flow logs to Splunk. 

1. Open the Splunk console and navigate to the Search tab within the Searching and Reporting app.
2. Run the following SPL query to view the ingested VPC Flow log records. Replace the placeholder in <> with the appropriate Splunk index name: 

index = <insert_index_name> sourcetype = “aws:cloudwatchlogs:vpcflow”

## Conclusion

This blog delved into how you can filter and ingest VPC flow logs into Splunk with the help of a Lambda function. VPC Flow logs was used as an example, but similar architectural pattern can be replicated for multiple log types stored in S3 buckets. The code example provides you with an extendable framework to ingest any AWS and non-AWS logs centralized in S3 into Splunk using the push-based mechanism. The filtering capability of the Lambda function can help you to ingest only the logs that you are interested in, thus helping to optimize costs by reducing the Splunk license utilization.

## Notice

This code interacts with Splunk which has terms published [here](https://www.splunk.com/en_us/legal/splunk-general-terms.html) and pricing described [here](https://www.splunk.com/en_us/products/pricing.html?utm_campaign=google_amer_en_search_brand&utm_source=google&utm_medium=cpc&utm_content=Splunk_Prod_Pricing_EN&utm_term=splunk%20pricing&_bk=splunk%20pricing&_bt=505717294225&_bm=e&_bn=g&_bg=43997963247&device=c&gclid=Cj0KCQjwmvSoBhDOARIsAK6aV7hdohCJwJNbtekZQuKF17FW_3zI3Pe4hfQKix-ScOusnmDRl1Zzs7gaAhwlEALw_wcB). You should be familiar with the pricing and confirm that your use case complies with the terms before proceeding.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

