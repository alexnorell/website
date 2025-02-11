---
title: "Add Cloudfront to a Website"
date: 2020-08-22T12:00:00-07:00
tags:
  - terraform
  - AWS
  - s3
  - cloudfront
categories:
  - infrastructure
draft: false
---

The domain I'm using, `norell.dev`, requires the site to be served via HTTPS, or else the browser won't display it. Google, the owners of the gTLD, requires HSTS for the domain.

From the [get.dev](https://get.dev/#benefits) page:

> Your security is our priority. The `.dev` top-level domain is included on the HSTS preload list, making HTTPS required on all connections to .dev websites and pages without needing individual HSTS registration or configuration. Security is built in.

Even though S3 is hosting the page, the browser will not display it due to HSTS. We can see that it is working using HTTP either with a browser that doesn't care about HSTS, or using curl.

```sh
$ curl www.norell.dev
<!DOCTYPE html>
<html>
    <head>
        <title>Test</title>
    </head>
    <body>
        <p>Test</p>
    </body>
</html>
```

S3 does not support hosting content with HTTPS, so I will need to set up CloudFront to act as an intermediary between the S3 bucket, and the requester. CloudFront is designed to make static content like a website available all over the world. It moves the content closer to the requester, giving the user a much better experience. It also has the benefit of allowing HTTPS support.


## Configuring Certificate

I first need to create a certificate to use for Cloudfront. For the certificate to work with CloudFront, it **must** be in `us-east-1`. If the certificate isn't in that region, CloudFront will not work. All of my website so far has been created in `us-west-1`, and I want to continue using that region. I need to create a separate region provider just for the certificate.

### US East Provider

In `main.tf`:

```tf
provider "aws" {
  version = "~> 3.2"
  region  = "us-east-1"
  alias   = "us_east_1"
}
```

### Certificate Creation

We will then be able to use this for the provider to create the certificate. The rest of the infrastructure will still be in `us-west-1`

In `cert.tf`:

```tf
resource "aws_acm_certificate" "www" {
  provider          = aws.us_east_1
  domain_name       = local.full_domain
  validation_method = "DNS"
  lifecycle {
    create_before_destroy = true
  }
}
```

### Certificate Validation

This will create a certificate in the `us-east-1` in Certificate Manager. This cert isn't usable until we verify that we own the domain.

```tf
resource "aws_route53_record" "www_validation" {
  for_each = {
    for dvo in aws_acm_certificate.www.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.domain.zone_id
}

resource "aws_acm_certificate_validation" "www" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.www.arn
  validation_record_fqdns = [for record in aws_route53_record.www_validation : record.fqdn]
}
```

This will set up the certification validation settings, and create the Route53 DNS entries to confirm that we are in control of the domain.

## CloudFront

Now that we have a certificate, we can create a CloudFront Distribution.

In `cloudfront.tf`:

```tf
resource "aws_cloudfront_distribution" "domain_distribution" {
  enabled             = true
  default_root_object = "​index.html"
  aliases             = [local.full_domain]
  origin {
    custom_origin_config {
      http_port              = "80"
      https_port             = "443"
      origin_protocol_policy = "http-only"
      origin_ssl_protocols   = ["TLSv1", "TLSv1.1", "TLSv1.2"]
    }
    domain_name = aws_s3_bucket.website.website_endpoint
    origin_id   = local.full_domain
  }
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.www.arn
    ssl_support_method  = "sni-only"
  }
  default_cache_behavior {
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = local.full_domain
    min_ttl                = 0
    default_ttl            = 86400
    max_ttl                = 31536000
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }
}
```

## Update Route53

We need to update the route in Route53 to send requests to CloudFront, rather than S3.

In `domain.tf`:

```tf
resource "aws_route53_record" "www" {
  zone_id = data.aws_route53_zone.domain.zone_id
  name    = local.full_domain
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.domain_distribution.domain_name
    zone_id                = aws_cloudfront_distribution.domain_distribution.hosted_zone_id
    evaluate_target_health = true
  }
}
