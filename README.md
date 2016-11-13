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

Wicked. Well shit I still haven't done anything.

Let's make a static website. Here's an index.html file as a landing page for a domain name I bought.

```html
<html>
  <head>
    <title>Heaviocity</title>
    <meta name="viewport" content="width=device-width">
  </head>
  <body>
    <style>
    body {
      background-color: rgb(21, 46, 201);
      color: white;
      font-family: sans-serif;
      text-align: center;
      padding: 200px;
    }
    </style>
    <h1>Heviocity</h1>
    <h5>Heaviness Maximus</h5>
  </body>
</html>
```

Let's create a bucket.

```sh
❯❯❯ aws s3api create-bucket --bucket heaviocity
/heaviocity
```

Then lets upload the index.html file:

```sh
❯❯❯ aws s3 cp heaviocity/index.html s3://heaviocity
upload: heaviocity/index.html to s3://heaviocity/index.html
```

Let's update the permissions so that anyone can read documents in this s3 bucket.

```sh
❯❯❯ aws s3api put-bucket-policy --bucket heaviocity --policy '{
  "Statement": [{
    "Action": ["s3:GetObject"],
    "Effect": "Allow",
	  "Principal": "*",
    "Resource": ["arn:aws:s3:::heaviocity/*"]
  }]
}'
```

To make a website, we're going to use the `website` command: http://docs.aws.amazon.com/cli/latest/reference/s3/website.html

Looks like they want an error.html file, so let's do that too:

```html
<html>
  <head>
    <title>404 Heaviocity</title>
    <meta name="viewport" content="width=device-width">
  </head>
  <body>
    <style>
    body {
      background-color: rgb(21, 46, 201);
      color: white;
      font-family: sans-serif;
      text-align: center;
      padding: 200px;
    }
    </style>
    <h1>Heviocity 404</h1>
    <h5><a href="/">Back to Safety!</a></h5>
  </body>
</html>
```

This time let's use sync:

```sh
❯❯❯ aws s3 sync heaviocity s3://heaviocity
upload: heaviocity/error.html to s3://heaviocity/error.html
```

Sweet. Now let's make it a website.

```sh
❯❯❯ aws s3 website s3://heaviocity/ --index-document index.html --error-document error.html
```

When you look up the bucket location:

```sh
❯❯❯ aws s3api get-bucket-location --bucket heaviocity
None
```

Apparently that defaults to `us-east-1` so you can actually find the website here: http://heaviocity.s3-website-us-east-1.amazonaws.com/

Looks like we could have used the `--create-bucket-configuration` option with `create-bucket` to put it in another location.

But if you want to distribute this around world in a speedy fashion, you'll want to use Cloudfront anyways.

But first, we should set up our domain name to point to this. I like to use Google Domains to register my domain name, and you'll need to use something like that. But then we can use Route 53 to handle everything else.

First, we need to create a new "hosted zone":

```sh
❯❯❯ aws route53 create-hosted-zone --name heaviocity.com --caller-reference chet
https://route53.amazonaws.com/2013-04-01/hostedzone/Z1XMR5FKGT8423
CHANGEINFO	/change/C2UHZ9ARRMWUL7	PENDING	2016-11-12T22:40:48.784Z
NAMESERVERS	ns-1313.awsdns-36.org
NAMESERVERS	ns-540.awsdns-03.net
NAMESERVERS	ns-1613.awsdns-09.co.uk
NAMESERVERS	ns-347.awsdns-43.com
HOSTEDZONE	chet	/hostedzone/Z1XMR5FKGT8423	heaviocity.com.	2
CONFIG	False

❯❯❯ aws route53 list-hosted-zones
HOSTEDZONES	chet	/hostedzone/Z1XMR5FKGT8423	heaviocity.com.	2
CONFIG	False

❯❯❯ aws route53 get-hosted-zone --id /hostedzone/Z1XMR5FKGT8423
NAMESERVERS	ns-1313.awsdns-36.org
NAMESERVERS	ns-540.awsdns-03.net
NAMESERVERS	ns-1613.awsdns-09.co.uk
NAMESERVERS	ns-347.awsdns-43.com
HOSTEDZONE	chet	/hostedzone/Z1XMR5FKGT8423	heaviocity.com.	2
CONFIG	False
```

Cool. Now we need to connect those name servers to the DNS website we used to register the domain. For me, thats Google Domains -- it was pretty easy to figure out.

You can check to see if the transfer worked:

```sh
❯❯❯ dig +recurse +trace heaviocity.com any
```

Didn't seem to work yet, but lets continue and see if it worked later (apparently this can take a day or two).

We need to create some DNS records to point to our s3 bucket.

```sh
❯❯❯ aws route53 list-resource-record-sets --hosted-zone-id /hostedzone/Z1XMR5FKGT8423
RESOURCERECORDSETS	heaviocity.com.	172800	NS
RESOURCERECORDS	ns-1313.awsdns-36.org.
RESOURCERECORDS	ns-540.awsdns-03.net.
RESOURCERECORDS	ns-1613.awsdns-09.co.uk.
RESOURCERECORDS	ns-347.awsdns-43.com.
RESOURCERECORDSETS	heaviocity.com.	900	SOA
RESOURCERECORDS	ns-1313.awsdns-36.org. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400
```

Lets use a CNAME to alias to our bucket: http://heaviocity.s3-website-us-east-1.amazonaws.com/

```sh
❯❯❯ aws route53 change-resource-record-sets --hosted-zone-id /hostedzone/Z1XMR5FKGT8423 --change-batch '{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "heaviocity.com",
        "Type": "A",
        "TTL": 60,
        "AliasTarget": {
          "HostedZoneId": "/hostedzone/Z1XMR5FKGT8423",
          "DNSName": "s3-website-us-east-1.amazonaws.com"
        },
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [
          {
            "Value": "heaviocity.com"
          }
        ]
      }
    }
  ]
}'
```

FUCK Nothing is working!

To clean up

```sh
aws route53 delete-hosted-zone --id /hostedzone/Z1XMR5FKGT8423
aws s3 rb s3://heaviocity --force
```

# S3 Round 2

```sh
# make bucket with the name of your website
aws s3 mb s3://heaviocity.com

# deploy html files
aws s3 sync ./heaviocity s3://heaviocity.com \
  --delete \
  --acl public-read \
  --include "*.html"

# THIS DOESNT WORK!!!
# but its the right idea...
mkdir -p .tmp
find . -iname '*.css' \
  -o -iname '*.js' \
  -o -iname '*.png' \
  -exec 'gzip -9 -n {} > .tmp' \;\
  -exec 'mv {}.gz .tmp/{}' \;

aws s3 sync ./heaviocity s3://heaviocity.com \
  --acl public-read
  --delete
  --include "*.js"
  --include "*.css"
  --include "*.png"
  --cache-control 'max-age=31536000' # cache for an entire year
  --content-encoding 'gzip'
```

Continue here: https://brandur.org/aws-intrinsic-static

- TODO
  - figure out how to handle DNS stuff from the cli
    - set up A record to point to s3
    - set up CNAME to alias www to @.
  - set up cloudfront
  - deployment
  - set up caching headers
  - gzip assets
  - set up logs
  - set up HTML5 routing
  - https setup

# EC2

- TODO
  - create an ec2 instance
    - ssh and play around
  - create a node app
    - serve from that server
  - api gateway / reverse proxy
  - create the node app in a docker container
    - deploy the docker container
  - send logs somewhere
  - restart on exit
  - simple deployment script / coordination
  - send email / text / notify on errors or events
  - health check

# TODO

- DNS stuff
  - configure subdomains
  - reverse proxying
- setup lambda on api gateway
- IoT chat application?

# EC2

http://docs.aws.amazon.com/cli/latest/userguide/tutorial-ec2-ubuntu.html

# Examples

- explain / understand
  - iam, s3, cloudfront, cloudformation, s3, dynamo, rds, sqs, lambda, iot, kinesis, redshift

- create and serve a static website from S3
- create a server-rendered Node.js website on Lambda
- create a website with user authentication using IAM
- send an email from a website
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
