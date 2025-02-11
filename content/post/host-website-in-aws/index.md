---
title: "Host Website in AWS S3"
date: 2020-08-18T23:06:28-07:00
tags:
  - terraform
  - AWS
  - s3
categories:
  - infrastructure
draft: false
---

I want to deploy a copy of this website to AWS S3, and set up all of the necessary infrastructure using terraform. This post will go over the steps that were taken to achieve this.

## Set up provider

This will be using AWS, so the terraform provider must be set. This will use the latest terraform version, `0.13`.

The provider is defined below, and located in `main.tf`

```tf
provider "aws" {
  version = "~> 3.0"
  region  = "us-west-1"
}
```

## Set up backend

This project will use S3 as the backend for storing the terraform state files. I created an S3 bucket using the AWS cli.

```sh
aws s3 mb s3://alexnorell-tfstates --region us-west-1
```

I then added the following to `backend.tf`

```tf
terraform {
  backend "s3" {
    bucket = "alexnorell-tfstates"
    key    = "website"
    region = "us-west-1"
  }
}
```

## Add s3 bucket

I need a bucket to put the static website in. I added the s3 bucket configuration to `website.tf`

```tf
resource "aws_s3_bucket" "website" {
  bucket = "website.norell.dev"
  acl    = "public-read"
  versioning {
    enabled = false
  }
  website {
    index_document = "index.html"
    error_document = "404.html"
  }
}
```

## Configure S3 Bucket to be public

While the bucket has been given the `public-read` ACL, from the [canned ACL list](https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html#canned-acl), the IAM policy permissions haven't been set up. To do this, I'll create an IAM policy document.


```tf
data "aws_iam_policy_document" "website_read_access" {
  statement {
    sid       = "PublicReadGetObject"
    effect    = "Allow"
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.website.arn}/*"]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
  }
}
```

This is the same as the standard JSON policy, but it fits into the terraform flow a lot better.

This policy then needs to be applied to the bucket.

```tf
resource "aws_s3_bucket_policy" "website_policy" {
  bucket = aws_s3_bucket.website.id
  policy = data.aws_iam_policy_document.website_read_access.json
}
```

Now anything in the bucket will be public.
