---
title: "Add External Domain in AWS and create record for S3"
date: 2020-08-19T21:21:09-07:00
lastmod: 2020-08-20T23:30:00-7:00
tags:
  - aws
  - route53
  - domain name
  - s3
categories:
  - infrastructure
draft: false
---

I have a domain name, `norell.dev` that is registered outside of AWS. I would like to use it for my development within AWS, but Amazon doesn't support the `.dev` domain name. Google is the gTLD owner, and Amazon and Google don't play well together. Even though Amazon won't let me transfer my domain to be registered in Route53, I can still configure it to be used using Hosted Zones.

I will follow [this guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/migrate-dns-domain-inactive.html) to get it set up.

## Steps

1. Log into AWS Route53
2. Add Hosted Zone
3. Log into registrar (Namecheap, GoDaddy, Google, etc.)
4. Change domain specific name server to the ones provided by AWS Route53
5. Wait 24-48 hours if you had DNSSEC enabled

## Configure website to use domain

Now that the domain is in Route53, I can set the domain for use by the website hosted in S3. You can find more information on that from the [previous post](../host-website-in-aws).

### Get Hosted Zone information

I don't want to have terraform manage the top level hosted zone, so I will use a `data` type to retrieve the information from Route53. In `domain.tf` I added:

```tf
variable "domain_name" {
    description = "domain name to use from route53"
    default = "norell.dev"
}

variable "subdomain" {
    description = "subdomain to create record and bucket for"
    default = "www"
}

data "aws_route53_zone" "domain" {
  name = var.domain_name
}
```

I want to make this generic, so I created variables, one for the generic top level domain name, and one for the subdomain. In this case, it is `norell.dev` and `www`.

### Reconfigure S3 Bucket

I created the S3 bucket for this website with the incorrect name. It needs to be **exactly** the same name as the domain name for Route53 to route to it. Since we have a variable for the subdomain now, and we have a data object for the domain hosted zone, we can use those to define the bucket name.


In `website.tf`:

```tf
resource "aws_s3_bucket" "website" {
  bucket = "${var.subdomain}.${data.aws_route53_zone.domain.name}"
...
```

### Create the alias

To add the alias for the bucket, I added the alias definition for the subdomain

Adding to `domain.tf`

```tf
resource "aws_route53_record" "www" {
  zone_id = data.aws_route53_zone.domain.zone_id
  name    = "${var.subdomain}.${data.aws_route53_zone.domain.name}"
  type    = "A"

  alias {
    name                   = aws_s3_bucket.website.website_domain
    zone_id                = aws_s3_bucket.website.hosted_zone_id
    evaluate_target_health = true
  }
}
```

This grabs the domain and hosted zone ID from the bucket, allowing for no hardcoded values for the alias.
