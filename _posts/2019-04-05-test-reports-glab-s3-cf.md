---
layout: post
title: 'Publishing private test reports using Gitlab, S3 and CloudFlare Access'
author: maido
categories: [tech, test, aws, cloudflare, gitlab]
image: /assets/images/testreports.png
featured: false
hidden: false
---

[GitLab](https://gitlab.com/), similarly to [GitHub](https://github.com/) has a useful functionality called [GitLab Pages](https://docs.gitlab.com/ee/user/project/pages/) which lets you publish a static website per repository. It's mostly used to publish simple blogs or webpages using static site builders like [Jekyll](https://jekyllrb.com/).

Since most of our repositories don't have anything to publish (expect for our [website](https://producement.com), [blog](https://blog.producement.com) and a couple of [Bootstrap](https://getbootstrap.com/) theme [examples](https://producement.gitlab.io/dashkit-theme/)) we thought that maybe we can use this functionality to publish our test and coverage reports instead!
Unfortunatrely one of the biggest limitations of the GitLab Pages functionality is that everything that you publish will be public, even if the repository itself is private. There is actually a [plan](https://gitlab.com/gitlab-com/gl-infra/infrastructure/issues/5576) that would fix this issue, but as of writing this blog post it has not been released to [GitLab.com](https://gitlab.com).

We don't want the test reports to be public because for example coverage reports contain a lot of source code we would not want to make public, so we had to look for an alternative. [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/Welcome.html) to the rescue!

## Setting up the S3 bucket

If you want to create an S3 bucket that is not publicly available, the easiest way probably would be is to create a [CloudFront](https://aws.amazon.com/cloudfront/) distribution that uses this bucket and add a [Lambda@Edge](https://aws.amazon.com/lambda/edge/) function that will set up [HTTP basic access authentication](https://en.wikipedia.org/wiki/Basic_access_authentication).
We decided that we didn't want to be bothered with credentials all the time and since we use [CloudFlare Access](https://www.cloudflare.com/products/cloudflare-access/) for other services, why not use that instead? 

Setting all of this up takes a few steps.

### DNS setup

First we need to create a DNS CNAME entry in CloudFlare for the bucket. Think of a name you want to use for the reports. For our example let's pick `reports.example.com`.

Create a CNAME record that is an alias for S3:

`reports.example.com CNAME reports.example.com.s3.amazonaws.com`

Leave the HTTP Proxy turned on.

### Create an S3 bucket

Important thing when creating the S3 bucket is to use exactly the same name for the bucket as your domain name. Otherwise your domain will not work!

You also need to enable static hosting for the bucket: 

**Properties -> Static web hosting -> Use this bucket to host a website**.

### Create an IAM policy to upload reports

We need to have an [AWS Access Key](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) to upload test reports to this bucket, so we need to create a policy that we can later add to the [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html) we are going to use.

The policy should look like this:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::reports.example.com/*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "*"
        }
    ]
}
```

Add this policy to a IAM User (or better, to an IAM Role) and create an access key that we can later add to the publisher configuration.

### S3 Bucket policy

By default the S3 bucket will not be publicly available (sensible defaults, yay!) and we don't really want to make it publicly available, but we need to access it somehow.
This is where we can use CloudFlare Access: we will only allow access from [CloudFlare IP's](https://www.cloudflare.com/ips/).

Create a new bucket policy for our bucket under **Permissions -> Bucket Policy**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CFAccess",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::reports.example.com/*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": [
                        "103.21.244.0/22",
                        "103.22.200.0/22",
                        "103.31.4.0/22",
                        "104.16.0.0/12",
                        "108.162.192.0/18",
                        "131.0.72.0/22",
                        "141.101.64.0/18",
                        "162.158.0.0/15",
                        "172.64.0.0/13",
                        "173.245.48.0/20",
                        "188.114.96.0/20",
                        "190.93.240.0/20",
                        "197.234.240.0/22",
                        "198.41.128.0/17",
                        "2400:cb00::/32",
                        "2405:8100::/32",
                        "2405:b500::/32",
                        "2606:4700::/32",
                        "2803:f800::/32",
                        "2c0f:f248::/32",
                        "2a06:98c0::/29"
                    ]
                }
            }
        }
    ]
}
```

Now we can access our bucket using our domain name, but it is still publicly available!

**NB!** It might take some time before everything will start to work together, be patient!

### Configuring CloudFlare Access

When CloudFlare Access is enabled and login methods are configured, only thing left to do is to add an Access Policy to your `reports.example.com` domain. 

**NB!** Don't forget this part, without this all the setup is meaningless and your bucket will be publicly available!

### Publishing test reports

Add your previously generated AWS Access Key to GitLab 

**Settings -> CI/CD -> Environment variables** as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

Example `.gitlab-ci.yml` configuration file for a [Gradle](https://gradle.org/) project with [JUnit](https://junit.org/junit5/) and [Jacoco](https://www.jacoco.org/jacoco/):

```yml
image: alpine:latest

variables:
  AWS_DEFAULT_REGION: eu-central-1 # The region of our S3 bucket
  BUCKET_NAME: reports.example.com  # Your bucket name

stages:
  - build
  - deploy

build:
  image: java:8-alpine

  stage: build
  script: ./gradlew check
  artifacts:
    when: always
    paths:
      - build/reports/tests/test
      - build/jacocoHtml

test-report:
  image: maidok/awscli:latest
  stage: deploy
  script:
    # Simple HTML page that has links to test and coverage reports
    - aws s3 cp etc/index.html s3://${BUCKET_NAME}/${CI_PROJECT_NAME}/${CI_COMMIT_REF_SLUG}/index.html
    - aws s3 cp build/reports/tests/test s3://${BUCKET_NAME}/${CI_PROJECT_NAME}/${CI_COMMIT_REF_SLUG}/test --recursive
    - aws s3 cp build/jacocoHtml s3://${BUCKET_NAME}/${CI_PROJECT_NAME}/${CI_COMMIT_REF_SLUG}/coverage --recursive
  environment:
    name: ${CI_COMMIT_REF_SLUG}-reports
    url: https://${BUCKET_NAME}/${CI_PROJECT_NAME}/${CI_COMMIT_REF_SLUG}/index.html
  when: always
```

Now after the build, even if it fails, a new environment is created under **Operations -> Environments**. If you open the link, CloudFlare will ask you for your credentials (if you are not already logged in) and you can see your reports!

### Conclusion

In this blog post I demonstrater a way how to publish static content using any CI server that uses [Docker](https://www.docker.com/) to an S3 bucket and serve it securely using CloudFlare Access. We are using it for reports, but this can be used for anything really, of course it is much simpler if you can leave things public.
