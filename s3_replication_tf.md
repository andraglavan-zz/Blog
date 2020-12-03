# Terraform S3 Cross Region Replication: from an unencrypted bucket to an encrypted bucket

I've been working with Terraform for a few months now, and one of the scenarios that I've encountered, that put me in trouble was this:
New client wants to migrate several buckets from the existing account, Ohio region, to the new account, Frankfurt region. This is, of course, no problem for AWS, and this type of migration can be found in a lot of scenarios already explained on the internet. But what was new was that some of the buckets were not encrypted at the source, and at the destination everything must be encrypted to comply with security standards. 

## What is the issue?
For the Cross Region Replication (CRR) to work, we need to do the following:
 * Enable Versioning for both buckets
 * At Source: Create an IAM role to handle the replication 
 * Setup the Replication for the source bucket 
 * At Destination: Accept the replication 

If both buckets have the encryption enabled, things will go smoothly. Same way it goes if both are unencrypted.
But if the Source bucket is unencrypted and the Destination bucket uses AWS KMS customer master keys (CMKs) to encrypt the Amazon S3 objects, things get a bit more interesting.

## What is the solution?
One of the best advices I have received while working with software for infrastructure as code in AWS, was that if I am going to deploy something new and have troubles with it, one good way to solve it is to go into the AWS console, and try to manually create what I need. This makes things clearer and helps to understand better what it's needed and how it needs to be modified in order to make it work.\
This was the process I followed, and after a few hours of trials and a support ticket with AWS, this was solved with the feedback that, this scenario is 'tricky'.\
The 2 things that must be done, in order to make the CRR work between an unencrypted Source bucket to an encrypted Destination bucket are: After the replication role is created 
 * In the Source account, get the role ARN and use it to create a new policy. This policy needs to be added to the KMS key in the Destination account.
 * Modify the role to add a new policy to it, to be able to use the KMS key in the Destination account. For this, the KMS key ARN is needed and the policy will look like this:
 ```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:Encrypt",
                "kms:GenerateDataKey*",
                "kms:ReEncrypt*",
                "kms:DescribeKey"
                ],
            "Resource": "arn:aws:kms:[aws-region]:[account-id]:key/1234abcd-12ab-34cd-56ef-1234567890ab"
        }
    ]
}
```
## How do I put it in code?

Let's say that the bucket to be replicated is called: **source-test-replication**, and it is in the Source account, in the Ohio region. The versioning is enabled, and the default encryption is disabled. The bucket in the Destination account is **destination-test-replication**.

The Terraform code for the normal replication, that creates a KMS key for the new bucket, includes these KMS resources:

 ```
resource "aws_kms_key" "replication_s3_kms_key" {
  description = "s3 encryption key"
}

resource "aws_kms_alias" "replication_s3_kms_alias" {
  name          = "alias/replication-s3-key"
  target_key_id = aws_kms_key.replication_s3_kms_key.key_id
}
 ```
For this scenario to work, the code needs to me modified and the following information need to be added:

 ```
data "aws_iam_policy_document" "kms_policy" {
   statement {
    sid = "Enable IAM User Permissions"
    effect = "Allow"
    principals {
      type = "AWS"
      identifiers = [
        "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        ]
    }
    actions = [
      "kms:*"
    ]
    resources = [
      "*"
    ]
  }
  statement {
    sid = "Allow use of the key"
    effect = "Allow"
    principals {
      type = "AWS"
      identifiers = [
        "arn:aws:iam::<Replication role ARN>”
        ]
    }
    actions = [
      "kms:Encrypt",
      "kms:Decrypt",
      "kms:ReEncrypt*",
      "kms:GenerateDataKey*",
      "kms:DescribeKey"
    ]
    resources = [
      "arn:aws:s3:::destination-test-replication"
    ]
  }
}

resource "aws_kms_key" "replication_s3_kms_key" {
  description = "s3 encryption key"
  policy = data.aws_iam_policy_document.kms_policy.json
}

resource "aws_kms_alias" "replication_s3_kms_alias" {
  name          = "alias/replication-s3-key"
  target_key_id = aws_kms_key.replication_s3_kms_key.key_id
}

```
Both statements are needed, and if you are getting any errors saying something like this:
```
Error: MalformedPolicyDocumentException: The new key policy will not allow you to update the key policy in the future.
```
it means that the first statement is missing.

This is all that needs to be done in code, but don't forget about the second requirement: the policy in the Source account to add to the replication role. For this we need to create this  new policy, chose a name, and attach it to the replication role:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:Encrypt",
                "kms:GenerateDataKey*",
                "kms:ReEncrypt*",
                "kms:DescribeKey"
                ],
            "Resource": "arn:aws:kms:us-east-2:<Source Account ID>:key/523b2035-e947-4c71-8690-db6b43589c34"
        }
    ]
}
```
To wrap it up, for the replication to work in this scenario, the KMS key in the Destination account needs to have a policy to allow the replication IAM role to use it, and the replication role needs to have a policy to use the KMS key in the destination account.
## Looking forward

This year at **re:Invent**, a lot of great things were announced for S3 and I am looking forward to seeing which one will facilitate the automated deployments and which one will be, let's say, a bit tricky to play with.
For a top of the S3 announcements at the event, please check this great article: https://www.sentiatechblog.com/aws-reinvent-2020-day-1-s3-announcements