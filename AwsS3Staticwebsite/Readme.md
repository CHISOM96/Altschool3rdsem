# Deploy a Static Website on AWS using Terraform

## Objective:
Create an AWS infrastructure to host a static website using Terraform. The infrastructure will include AWS S3 for storing the website files, CloudFront for content delivery, and Route 53 for domain name management. Additional configurations will involve setting up IAM roles and policies, API Gateway, and SSL certificates.
## Prerequisites:
- AWS Account
- Domain name registered in Route 53
- Terraform installed on your local machine
## Setting up the AWS account
I decided to setup awscli on my local machine before beginning the project. Steps:
1. Download awscli by running the following command.
```
sudo apt-get update
sudo apt-get install -y awscli
```
2. Verify install by running:
```
aws --version
```
Response should be similar to this;
```
aws-cli/1.22.34 Python/3.10.12 Linux/6.2.0-26-generic botocore/1.23.34
```
3. Configure AWS CLI. Prior to doing this I had setup an account with the needed permisssions,
```
aws configure
```
## Tasks
1. Create the necessary files to match the tree
  * Create Folder for Project
  ```
  mkdir <Folder Name> && cd <Folder Name>
  ```
  * Create main.tf
  ```
  cat <<EOF > main.tf
  module "s3_bucket" {
    source = "./modules/s3"
    name   = var.s3_bucket_name
  }

  module "cloudfront" {
    source          = "./modules/cloudfront"
    s3_bucket_name  = var.s3_bucket_name
    domain_name     = var.domain_name
    certificate_arn = var.certificate_arn
  }

  module "route53" {
    source      = "./modules/route53"
    domain_name = var.domain_name
    cf_domain   = module.cloudfront.cf_domain_name
  }
  EOF
  ```
  * Create Modules folder
  ```
  mkdir modules && cd modules && mkdir s3 && mkdir cloudfro
  nt && mkdir route53
  ```
  * Create S3 Module
  ```
  cat <<EOF > s3/main.tf
  resource "aws_s3_bucket" "static_site" {
    bucket = var.name
    acl    = "public-read"

    website {
      index_document = "index.html"
      error_document = "error.html"
    }
  }

  resource "aws_s3_bucket_policy" "static_site_policy" {
    bucket = aws_s3_bucket.static_site.id
    policy = data.aws_iam_policy_document.s3_bucket_policy.json
  }
  EOF
  ```
  * Create Cloudfront Module
  ```
  cat <<'EOF' > cloudfront/main.tf
  resource "aws_cloudfront_distribution" "cdn" {
    origin {
      domain_name = "${var.s3_bucket_name}.s3.amazonaws.com"
      origin_id   = var.s3_bucket_name

      s3_origin_config {
        origin_access_identity = aws_cloudfront_origin_access_identity.origin_access_identity.cloudfront_access_identity_path
      }
    }

    enabled             = true
    is_ipv6_enabled     = true
    comment             = "CDN for ${var.s3_bucket_name}"
    default_root_object = "index.html"

    viewer_certificate {
      acm_certificate_arn = var.certificate_arn
      ssl_support_method  = "sni-only"
    }

    default_cache_behavior {
      allowed_methods  = ["GET", "HEAD"]
      cached_methods   = ["GET", "HEAD"]
      target_origin_id = var.s3_bucket_name

      forwarded_values {
        query_string = false
        cookies {
          forward = "none"
        }
      }

      viewer_protocol_policy = "redirect-to-https"
      min_ttl                = 0
      default_ttl            = 3600
      max_ttl                = 86400
    }
  }

  output "cf_domain_name" {
    value = aws_cloudfront_distribution.cdn.domain_name
  }
  EOF
  ```
  * Create route53 module
  ```
  cat <<'EOF' > route53/main.tf
  resource "aws_route53_record" "www" {
    zone_id = aws_route53_zone.primary.zone_id
    name    = var.domain_name
    type    = "A"

    alias {
      name                   = var.cf_domain
      zone_id                = aws_cloudfront_distribution.cdn.hosted_zone_id
      evaluate_target_health = false
    }
  }

  data "aws_route53_zone" "primary" {
    name         = var.domain_name
    private_zone = false
  }
  EOF
  ```
  * Create init.tf file
  ```
  cat <<EOF > init.tf
  provider "aws" {
    region = var.region
  }
  EOF
  ```
  * Initialize terraform after returning to the root directory
  ```
  cd ..
  ```
  ```
  terraform init
  ```
  * Create variables.tf file


```
cat <<EOF > variables.tf
variable "region" {
  description = "AWS region"
  default     = "us-east-1"
}

variable "domain_name" {
  description = "The domain name to use for the static website"
  type        = string
}

variable "s3_bucket_name" {
  description = "Name of the S3 bucket to create"
  type        = string
}

variable "certificate_arn" {
  description = "ARN of the ACM certificate"
  type        = string
}
EOF
```
