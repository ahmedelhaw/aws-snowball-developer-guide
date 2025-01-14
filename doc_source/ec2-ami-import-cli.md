# Importing an Image into Your Device as an Amazon EC2 AMI<a name="ec2-ami-import-cli"></a>

Using the AWS CLI, you can use the AWS feature VM Import/Export to import images into your AWS Snowball Edge device as EC2 instances\. After you import an image, you register it as an Amazon Machine Image \(AMI\) and launch it as an Amazon EC2 instance\.

**Topics**
+ [Step 1: Prepare the Image](#prepare-image-cli)
+ [Step 2: Set Up Required Permissions](#setup-permission-cli)
+ [Step 3: Import the Snapshot to Amazon S3 on Your Device](#import-snapshot-cli)
+ [Step 4: Register the Snapshot as an AMI](#register-snapshot-cli)
+ [Step 5: Launch the AMI](#launch-ami-cli)
+ [Describe Import Tasks](#decribe-import-task-cli)
+ [Cancel an Import Task](#cancel-import-task-cli)
+ [Describe Snapshots](#describe-snapshots-cli)
+ [Delete a Snapshot from Your Device](#delete-snapshot-cli)
+ [Deregister an AMI](#deregister-snapshot-cli)

## Step 1: Prepare the Image<a name="prepare-image-cli"></a>

 You can either export an image from the AWS Cloud using VM Import/Export, or generate the image locally using your choice of virtualization platform\. You can also export your image from the AWS Cloud\.

 
+ To import an image from AWS using VM Import/Export, see [Importing a VM as an Image Using VM Import/Export](https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html)\.
+ To export an image from an AMI to AWS using VM Import/Export, see [Exporting a VM Directly from an Amazon Machine Image \(AMI\)](https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport_image.html)\.

**Note**  
Be aware of the following limitations when uploading images to a Snowball Edge device\.  
Snow Family devices currently only support importing snapshots that are in the RAW image format\. 
Snow Family devices currently only support importing snapshots with sizes from 1 GB to 1 TB\.

When you have the image, you must upload it to Amazon S3 on your device because you can only import images as snapshots from Amazon S3 that are available on your device or cluster\. During the importing process, you choose the S3 bucket on your Snowball Edge device to store the image in\. 

## Step 2: Set Up Required Permissions<a name="setup-permission-cli"></a>

For the import to be successful, you must set up permissions for VM Import/Export, Amazon EC2, and the user\.

**Note**  
 The service roles and policies that provide these permissions are located on the Snow Family device\.

### Permissions Required for VM Import/Export<a name="vmie-permissions"></a>

**Create a role**

Before you can start the import process, you must create an IAM role with a trust policy that allows Snowball VM Import/Export to assume the role\. Additional permissions are given to the role to allow VM Import/Export to access the image stored in the S3 bucket on the device\. 

The default role name for the API is `vmimport`\. You can change it by using the `--role-name` string in the command\.

**Example command**  

```
aws iam create-role --role-name vmimport --assume-role-policy-document file:///path/to/trust-policy.json --profile xx --endpoint http://10.1.2.3:6078 --region snow
```

Following is an example trust policy required to be attached to the role so that VM Import/Export can access the snapshot that needs to be imported from the S3 bucket\. 

```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Principal":{
            "Service":"vmie.amazonaws.com"
         },
         "Action":"sts:AssumeRole"
      }
   ]
}
```

The following is an example output from the `create-role` command\. 

```
{
        "Role": {
            "Path": "/",            
            "RoleName": "vmimport",            
            "RoleId": "XXXXXXXXXXXXXXXXXXXXXXXXXX",            
            "Arn": "arn:aws:iam::123456789012:role/vmimport",            
            "CreateDate": "2020-07-30T01:01:57.415000+00:00",            
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",                
                "Statement": [
                    {
                        "Effect": "Allow",                        
                        "Principal": {
                            "Service": "vmie.amazonaws.com"                    
                        },                        
                        "Action": "sts:AssumeRole"                    
                    }
                ]
            },            
            "MaxSessionDuration": 3600        
        }
 }
```

**Create a policy for the role and attach the policy to the role**

You now attach a policy to the preceding role and grant permissions to access the required resources\. This allows the local VM Import/Export service to download the snapshot from Amazon S3 on the device\. 

**Example command**  

```
aws iam create-policy --policy-name vmimport-resource-policy --policy-document file:///path/to/vmie-resource-policy.json --profile xxx --endpoint http://10.1.2.3:6078 --region snow
```
The following example policy has the minimum required permissions to access Amazon S3\. You can change the S3 bucket name if you need to\. You also can use prefixes to further narrow down the location where VM Import/Export can import snapshots from\.  

```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket",
            "s3:GetMetadata"
         ],
         "Resource":[
            "arn:aws:s3:::import-snapshot-bucket-name",
            "arn:aws:s3:::import-snapshot-bucket-name/*"
         ]
      }
   ]
}
```
The following is an example output from the `create-policy` command\.  

```
{
   "Policy":{
      "PolicyName":"vmimport-resource-policy",
      "PolicyId":"ANPACEMGEZDGNBVGY3TQOJQGEZAAAABOOEE3IIHAAAABWZJPI2VW4UUTFEDBC2R",
      "Arn":"arn:aws:iam::123456789012:policy/vmimport-resource-policy",
      "Path":"/",
      "DefaultVersionId":"v1",
      "AttachmentCount":0,
      "IsAttachable":true,
      "CreateDate":"2020-07-25T23:27:35.690000+00:00",
      "UpdateDate":"2020-07-25T23:27:35.690000+00:00"
   }
}
```
Attach the policy that you created in the previous step to the role that you created in step 2\.  

**Example command**  

```
aws iam attach-role-policy --role-name vmimport --policy-arn arn:aws:iam::123456789012:policy/vmimport-resource-policy --profile xxx --endpoint http://10.1.2.3:6078 --region snow
```

### Permissions Required by the Caller<a name="caller-permissions"></a>

In addition to the role for the Snowball Edge VM Import/Export to assume, you also must ensure that the user has the permissions that allow them to pass the role to VMIE\. If you use the default root user to perform the import, the root user already has all the permissions required, so you can skip this step, and go to step 3\.

Attach the following two IAM permissions to the user account that is doing the import\.
+ `pass-role`
+ `get-role`

**Example command**  

```
aws iam create-policy --policy-name caller-policy --policy-document file:///path/to/caller-policy.json --profile xx --endpoint http://10.1.2.3:6078 --region snow
```
The following is an example policy that allows a user to perform the `get-role` and `pass-role` actions for the IAM role\.   

```
{
   "Version":"2012-10-17",
   "Statement":[
        {
            "Effect":"Allow",
            "Action": "iam:GetRole",
            "Resource":"*"
        },
        {
            "Sid": "iamPassRole",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "importexport.amazonaws.com"
                }
            }
        }
   ]
}
```

**Example output**  

```
{
   "Policy":{
      "PolicyName":"caller-policy",
      "PolicyId":"ANPACEMGEZDGNBVGY3TQOJQGEZAAAABOOOTUOE3AAAAAAPPBEUM7Q7ARPUE53C6R",
      "Arn":"arn:aws:iam::123456789012:policy/caller-policy",
      "Path":"/",
      "DefaultVersionId":"v1",
      "AttachmentCount":0,
      "IsAttachable":true,
      "CreateDate":"2020-07-30T00:58:25.309000+00:00",
      "UpdateDate":"2020-07-30T00:58:25.309000+00:00"
   }
}
```
After the policy has been generated, attach the policy to the IAM users that will call the Amazon EC2 API or CLI operation to import the snapshot\.  

**Example command**  

```
aws iam attach-user-policy --user-name yourUser --policy-arn arn:aws:iam::123456789012:policy/caller-policy --profile xx --endpoint http://10.1.2.3:6078 --region snow
```

### Permissions Required to Call Amazon EC2 APIs on Your Device<a name="ec2-permissions"></a>

To import a snapshot, the IAM user must have the `ec2:ImportSnapshot` permissions\. If restricting access to the user is not required, you can use the `ec2:*` permissions to grant full Amazon EC2 access\. The following are the permissions that can be granted or restricted for Amazon EC2 on your device\. 

```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "ec2:ImportSnapshot",
            "ec2:DescribeImportSnapshotTasks",
            "ec2:CancelImportTask",
            "ec2:DescribeSnapshots",
            "ec2:DeleteSnapshot",
            "ec2:RegisterImage",
            "ec2:DescribeImages",
            "ec2:DeregisterImage"
         ],
         "Resource":"*"
      }
   ]
}
```

## Step 3: Import the Snapshot to Amazon S3 on Your Device<a name="import-snapshot-cli"></a>

The next step is to import your snapshot into the Amazon S3 on your device\.

**Example command**  

```
aws ec2 import-snapshot --disk-container "Format=RAW,UserBucket={S3Bucket=bucket,S3Key=vmimport/image01}" --profile xx --endpoint http://10.1.2.3:8008 --region snow
```

For more information, see [import\-snapshot](https://docs.aws.amazon.com/v2/documentation/api/latest/reference/ec2/import-snapshot.html)\.

This command doesn't support the following switches\.
+ \[\-\-client\-data `value`\] 
+ \[\-\-client\-token `value`\]
+ \[\-\-dry\-run\]
+ \[\-\-no\-dry\-run\]
+ \[\-\-encrypted\]
+ \[\-\-no\-encrypted\]
+ \[\-\-kms\-key\-id `value`\]
+ \[\-\-tag\-specifications `value`\]

**Example output**  

```
{
   "ImportTaskId":"s.import-snap-1234567890abc",
   "SnapshotTaskDetail":{
      "DiskImageSize":2.0,
      "Encrypted":false,
      "Format":"RAW",
      "Progress":"3",
      "Status":"active",
      "StatusMessage":"pending",
      "UserBucket":{
         "S3Bucket":"bucket",
         "S3Key":"vmimport/image01"
      }
   }
}
```
Snowball Edge currently only allows one active import job to run at a time, per device\. To start a new import task, either wait for the current task to finish, or choose another available node in a cluster\. You can also choose to cancel the current import if you want\. To prevent delays, don't reboot the Snowball Edge device while the import is in progress\. If you reboot the device, the import will fail, and progress will be deleted when the device becomes accessible\. To check the status of your snapshot import task status, use the following command:  

```
aws ec2 describe-snapshots --endpoint http://10.1.2.3:8008 --profile xx
```

## Step 4: Register the Snapshot as an AMI<a name="register-snapshot-cli"></a>

When the snapshot import to the device is successful, you can register it using the `register-image` command\. 

**Note**  
You can only register an AMI when all its snapshots are available\.

For more information, see [register\-image](https://docs.aws.amazon.com/v2/documentation/api/latest/reference/ec2/register-image.html)\.

**Example command**  

```
aws ec2 register-image \
--name ami-01 \
--description my-ami-01 \
--block-device-mappings "[{\"DeviceName\": \"/dev/sda1\",\"Ebs\":{\"SnapshotId\": \"s.snap-0371c902634195955\", \"VolumeType\":\"sbg1\"}}, {\"DeviceName\": \"/dev/sda2\",\"Ebs\":{\"VolumeType\":\"sbp1\", \"VolumeSize\":1000}}]" \
--root-device-name /dev/sda1 \
--profile xxx \
--endpoint http://10.1.2.3:8008
```

Block device mapping JSON \(unescaped\)

```
[
   {
      "DeviceName":"/dev/sda1",
      "Ebs":{
         "SnapshotId":"s.snap-0371c902634195955",
         "VolumeType":"sbg1"
      }
   },
   {
      "DeviceName":"/dev/sda2",
      "Ebs":{
         "VolumeType":"sbp1",
         "VolumeSize":1000
      }
   }
]
```

**Example output**  

```
{
    "ImageId": "s.ami-8de47d2e397937318"
 }
```

## Step 5: Launch the AMI<a name="launch-ami-cli"></a>

To launch an instance, see [run\-instances](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html)\.



```
aws ec2 run-instances --image-id s.ami-xxxxxxxxxxx --instance-type sbe-c.large --profile xx --endpoint http://10.1.2.3:8008
```

Example output

```
{
   "Instances":[
      {
         "SourceDestCheck":false,
         "CpuOptions":{
            "CoreCount":1,
            "ThreadsPerCore":2
         },
         "InstanceId":"s.i-12345a73123456d1",
         "EnaSupport":false,
         "ImageId":"s.ami-1234567890abcdefg",
         "State":{
            "Code":0,
            "Name":"pending"
         },
         "EbsOptimized":false,
         "SecurityGroups":[
            {
               "GroupName":"default",
               "GroupId":"s.sg-1234567890abc"
            }
         ],
         "RootDeviceName":"/dev/sda1",
         "AmiLaunchIndex":0,
         "InstanceType":"sbe-c.large"
      }
   ],
   "ReservationId":"s.r-1234567890abc"
}
```

## Describe Import Tasks<a name="decribe-import-task-cli"></a>

To see the current state of the import progress, you can run the Amazon EC2 `describe-import-snapshot-tasks` command\. This API supports pagination and filtering on the `task-state`\. 

**Example command**  

```
aws ec2 describe-import-snapshot-tasks --filter Name=task-state,Values=active --profile xx --endpoint http://10.1.2.3:8008
```

**Example output**  

```
{
        "ImportSnapshotTasks": [
            {
                "ImportTaskId": "s.import-snap-8f6bfd7fc9ead9aca",                
                "SnapshotTaskDetail": {
                    "Description": "Created by AWS-Snowball-VMImport service for s.import-snap-8f6bfd7fc9ead9aca",                    
                    "DiskImageSize": 8.0,                    
                    "Encrypted": false,                    
                    "Format": "RAW",  
                    "Progress": "3",                  
                    "SnapshotId": "s.snap-848a22d7518ad442b",                    
                    "Status": "active", 
                    "StatusMessage": "pending",                   
                    "UserBucket": {
                        "S3Bucket": "bucket1",                        
                        "S3Key": "image1"                        
                    }
                }
            }
        ]
 }
```

**Note**  
This API only shows output for tasks that have successfully completed or been marked as deleted within the last 7 days\. Filtering only supports `Name=task-state`, `Values=active | deleting | deleted | completed`

This command doesn't support the following switches\.
+ \[\-\-dry\-run\]
+ \[\-\-no\-dry\-run\]

## Cancel an Import Task<a name="cancel-import-task-cli"></a>

To cancel an import task, run the `cancel-import-task` command\. 

**Example command**  

```
aws ec2 cancel-import-task --import-task-id s.import-snap-8234ef2a01cc3b0c6 --profile xx --endpoint http://10.1.2.3:8008
```

**Example output**  

```
{
        "ImportTaskId": "s.import-snap-8234ef2a01cc3b0c6",
        "PreviousState": "active",
        "State": "deleting"
}
```
Only tasks that are not in a completed state can be canceled\.

This command doesn't support the following switches\.
+ \[\-\-dry\-run\]
+ \[\-\-no\-dry\-run\]

## Describe Snapshots<a name="describe-snapshots-cli"></a>

After a snapshot is imported, you can use this command to describe it\. To filter the snapshots, you can pass in `--snapshot-ids` with the snapshot ID from the previous import task response\. This API supports pagination and filter on `volume-id`, `status`, and `start-time`\.

**Example command**  

```
aws ec2 describe-snapshots --snapshot-ids s.snap-848a22d7518ad442b --profile xx --endpoint http://10.1.2.3:8008
```

**Example output**  

```
{
    "Snapshots": [
        {
            "Description": "Created by AWS-Snowball-VMImport service for s.import-snap-8f6bfd7fc9ead9aca",
            "Encrypted": false,
            "OwnerId": "123456789012",
            "SnapshotId": "s.snap-848a22d7518ad442b",
            "StartTime": "2020-07-30T04:31:05.032000+00:00",
            "State": "completed",
            "VolumeSize": 8
        }
    ]
 }
```

This command doesn't support the following switches\.
+ \[\-\-restorable\-by\-user\-ids `value`\] 
+ \[\-\-dry\-run\]
+ \[\-\-no\-dry\-run\]

## Delete a Snapshot from Your Device<a name="delete-snapshot-cli"></a>

To remove snapshots that you own and you no longer need, you can use the `delete-snapshot` command\. 

**Example command**  

```
aws ec2 delete-snapshot --snapshot-id s.snap-848a22d7518ad442b --profile XXX --endpoint http://012.345.6.78:8008 --region snow
```

**Note**  
Snowball Edge does not support deleting snapshots that are in a **PENDING** state or if it is designated as a root device for an AMI\.

This command doesn't support the following switches\. 
+ \[\-\-dry\-run\]
+ \[\-\-no\-dry\-run\]

## Deregister an AMI<a name="deregister-snapshot-cli"></a>

To deregister AMIs that you no longer need, you can run the `deregister-image` command\. Deregistering an AMI that is in the **Pending** state is not currently supported\.

**Example command**  

```
aws ec2 deregister-image --image-id ami-01 --profile xxx --endpoint http://10.1.2.3:8008
```

This command doesn't support the following switches\.
+ \[\-\-dry\-run\]
+ \[\-\-no\-dry\-run\]