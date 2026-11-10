# Securing a Static Website on AWS S3

Hands-on project: I deployed a static site on S3 the insecure way, then rebuilt it securely with CloudFront and Origin Access Control (OAC), and documented the difference.

**Services used:** S3, CloudFront, IAM (bucket policies)
**Region:** eu-north-1 (Stockholm)

## 1. Insecure version (what many real-world leaks look like)

**Setup**
- Created an S3 bucket and uploaded `index.html`
- Turned **Block Public Access off**
- Added a bucket policy allowing anyone to read every object:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<BUCKET-NAME>/*"
  }]
}
```

**Result:** the page loaded for anyone at the S3 URL.
Screenshot: `screenshots/01-public-page.png`, `screenshots/02-public-policy.png`

**Risks**
- Anyone on the internet can read every object in the bucket
- Adding a sensitive file later would expose it immediately
- No access logging, no way to restrict who reads what
- I also noticed the legacy ACL screen can grant read access to "any AWS account holder", another common misconfiguration

## 2. Secure version

**Changes**
1. Turned **Block Public Access on** (all four settings)
2. Deleted the public bucket policy, so the direct S3 URL returned **Access Denied**
   Screenshot: `screenshots/03-access-denied.png`
3. Created a CloudFront distribution with the S3 bucket as origin, using **Origin Access Control (OAC)**
4. Set the default root object to `index.html` and left the origin path empty
5. Applied a bucket policy that only lets my CloudFront distribution read objects:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<BUCKET-NAME>/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT-ID>:distribution/<DISTRIBUTION-ID>"
      }
    }
  }]
}
```

**Result:** the site loads over HTTPS at the CloudFront URL, while the bucket stays private.
Screenshot: `screenshots/04-cloudfront-page.png`, `screenshots/05-secure-policy.png`

## 3. Before vs after

| | Insecure | Secure |
|---|---|---|
| Bucket access | Public to everyone | Private |
| Block Public Access | Off | On |
| Who can read objects | Anyone | Only my CloudFront distribution |
| Direct S3 URL | Works | Access Denied |
| Delivery | Direct from S3 | CloudFront (HTTPS, caching) |

## 4. What I learned
- Bucket policies with `Principal: "*"` are the pattern that causes public S3 exposure
- Block Public Access acts as a safety net on top of policies
- OAC lets CloudFront read a private bucket, and the `SourceArn` condition limits access to one specific distribution
- Mistakes I hit and fixed: a wrong origin path (`/index.html` instead of empty), and an empty `index.html` file

## 5. Next steps
- Launch an EC2 instance and practice security group hardening
- Build a mini-CSPM in Python (boto3) that automatically detects these misconfigurations
- Map checks to the CIS AWS Foundations Benchmark
