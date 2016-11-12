# Setup

```sh
brew install awscli
```

Go here and create some access keys: https://console.aws.amazon.com/iam/home#/security_credential

```sh
❯❯❯ aws configure
AWS Access Key ID [None]: XXX
AWS Secret Access Key [None]: XXX
Default region name [None]: us-west-1
Default output format [None]: text
```

# S3

List your buckets, here's some random shit from my struggling through the aws dashboard.

```sh
❯❯❯ aws s3 ls
2016-03-12 18:20:26 elasticbeanstalk-us-east-1-368059788469
2015-03-30 14:57:00 elasticbeanstalk-us-west-2-368059788469
```

Here's some helpful documentation about the cli: http://docs.aws.amazon.com/cli/latest/reference/s3/index.html#cli-aws-s3

```sh
❯❯❯ aws s3 ls s3://elasticbeanstalk-us-east-1-368059788469
```

Empty! Lets delete it!

There's a different set of meta commands for handling the buckets: https://docs.aws.amazon.com/cli/latest/reference/s3api/index.html#cli-aws-s3api

```sh
❯❯❯ aws s3api delete-bucket --bucket elasticbeanstalk-us-east-1-368059788469

An error occurred (AccessDenied) when calling the DeleteBucket operation: Access Denied
```

Oy. AWS already being a bitch. Lets see what we can find:

```sh
❯❯❯ aws s3api get-bucket-policy --bucket elasticbeanstalk-us-east-1-368059788469
{"Version":"2008-10-17","Statement":[{"Sid":"eb-ad78f54a-f239-4c90-adda-49e5f56cb51e","Effect":"Allow","Principal":{"AWS":"arn:aws:iam::368059788469:role/aws-elasticbeanstalk-ec2-role"},"Action":"s3:PutObject","Resource":"arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469/resources/environments/logs/*"},{"Sid":"eb-af163bf3-d27b-4712-b795-d1e33e331ca4","Effect":"Allow","Principal":{"AWS":"arn:aws:iam::368059788469:role/aws-elasticbeanstalk-ec2-role"},"Action":["s3:ListBucketVersions","s3:ListBucket","s3:GetObjectVersion","s3:GetObject"],"Resource":["arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469/resources/environments/*","arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469"]},{"Sid":"eb-58950a8c-feb6-11e2-89e0-0800277d041b","Effect":"Deny","Principal":{"AWS":"*"},"Action":"s3:DeleteBucket","Resource":"arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469"}]}
```

Lets pretty print that:

```sh
❯❯❯ aws s3api get-bucket-policy --bucket elasticbeanstalk-us-east-1-368059788469 | python -m json.tool
{
    "Statement": [
        {
            "Action": "s3:PutObject",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::368059788469:role/aws-elasticbeanstalk-ec2-role"
            },
            "Resource": "arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469/resources/environments/logs/*",
            "Sid": "eb-ad78f54a-f239-4c90-adda-49e5f56cb51e"
        },
        {
            "Action": [
                "s3:ListBucketVersions",
                "s3:ListBucket",
                "s3:GetObjectVersion",
                "s3:GetObject"
            ],
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::368059788469:role/aws-elasticbeanstalk-ec2-role"
            },
            "Resource": [
                "arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469/resources/environments/*",
                "arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469"
            ],
            "Sid": "eb-af163bf3-d27b-4712-b795-d1e33e331ca4"
        },
        {
            "Action": "s3:DeleteBucket",
            "Effect": "Deny",
            "Principal": {
                "AWS": "*"
            },
            "Resource": "arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469",
            "Sid": "eb-58950a8c-feb6-11e2-89e0-0800277d041b"
        }
    ],
    "Version": "2008-10-17"
}
```

Ok. So we're not letting anyone delete the bucket so let's fix that.

```sh
❯❯❯ aws iam get-user
USER	arn:aws:iam::368059788469:root	2015-03-30T18:49:51Z	2016-11-12T20:14:26Z	368059788469
```

So it looks like that's my user name: `arn:aws:iam::368059788469:root`. Now we want to create a policy that let's me delete this damn thing.

https://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-policy.html

```sh
❯❯❯ aws s3api put-bucket-policy --bucket elasticbeanstalk-us-east-1-368059788469 --policy '{
  "Statement": [
    {
      "Action": "s3:DeleteBucket",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::368059788469:root"
      },
      "Resource": "arn:aws:s3:::elasticbeanstalk-us-east-1-368059788469"
    }
  ]
}'
```

Alright, now let's delete it.

```sh
❯❯❯ aws s3api delete-bucket --bucket elasticbeanstalk-us-east-1-368059788469
```

I can't believe that worked, and I can't believe that took so long. But at least its better than using that terrible GUI!

```
❯❯❯ aws s3 ls
2015-03-30 14:57:00 elasticbeanstalk-us-west-2-368059788469
```

Now for the other one:

```sh
❯❯❯ aws s3api delete-bucket --bucket elasticbeanstalk-us-west-2-368059788469

An error occurred (BucketNotEmpty) when calling the DeleteBucket operation: The bucket you tried to delete is not empty
```

Interesting, apparently this is from an old college project:

```sh
❯❯❯ aws s3 ls s3://elasticbeanstalk-us-west-2-368059788469
2015-03-30 14:56:53          0 .elasticbeanstalk
2015-03-30 15:03:38       3726 2015089mTf-hw6.php.zip
2015-04-02 17:46:16       7448 2015092Amu-index.zip
2015-04-02 17:50:51       7449 2015092RhK-hw8.zip
2015-04-02 17:48:21       7448 2015092U9K-hw8.zip
```

Let's delete it all:

```sh
aws s3 rm s3://elasticbeanstalk-us-west-2-368059788469 --recursive
delete: s3://elasticbeanstalk-us-west-2-368059788469/.elasticbeanstalk
delete: s3://elasticbeanstalk-us-west-2-368059788469/2015092RhK-hw8.zip
delete: s3://elasticbeanstalk-us-west-2-368059788469/2015092Amu-index.zip
delete: s3://elasticbeanstalk-us-west-2-368059788469/2015089mTf-hw6.php.zip
delete: s3://elasticbeanstalk-us-west-2-368059788469/2015092U9K-hw8.zip
```

Then let's delete the bucket:

```sh
❯❯❯ aws s3api delete-bucket --bucket elasticbeanstalk-us-west-2-368059788469
```

Wicked.

# Examples

- serve a static website from Lambda
- send an email
- create a database and do pixel tracking with emails
- create a database and signup for email list


# Blah

```sh
brew install aws-shell
```

OR create a config file in `~/.aws/config`

```
[default]
aws_access_key_id = XXX
aws_secret_access_key = XXX
region = us-west-1
output = text

[profile chet]
aws_access_key_id = XXX
aws_secret_access_key = XXX
region = us-west-1
output = text
```

```sh
export AWS_CONFIG_FILE=$HOME/.aws/config
export AWS_DEFAULT_PROFILE=chet
```

https://forums.aws.amazon.com/message.jspa?messageID=520964
