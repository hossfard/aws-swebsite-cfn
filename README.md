# Synopsis

Cloudformation templates for creating infrastructure for static site
hosting with HTTPS support on AWS.

```text
  [Your TLD provider] --> [AWS Route53] --> [AWS Cloudfront] + [Certificate] -> [S3 URL]
```

This assumes that you already own a domain. AWS Route53 DNS can be
skipped altogether if using a subdomain.


# Requirements

- [awscli](https://aws.amazon.com/cli/)

# TLDR Quick guide

1. Create certificate stack

```bash
aws --profile <profile-name> \
    cloudformation create-stack \
    --stack-name <cert-stack-name> \
    --region "us-east-1" \
    --template-body file://./src/cert.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=VerificationDomain,ParameterValue="<example.com>"
# parameters are defined inside the config yaml file
```

where
 - `<profile-name>` is the profile name under `~/.aws/credentials`
    with appropriate permissions for creating cloudformation stacks, S3
    buckets, Cloudfront distribution, and Certificate.
 - `<cert-stack-name>` name assigned to the stack
 - `<sub.example.com>` is the domain name e.g. `example.com` or
   `sub.example.com`
 -  AWS will send emails to `<example.com>` domains for verification

 ```text
 administrator@<example.com>
 hostmaster@<example.com>
 postmaster@<example.com>
 webmaster@<example.com>
 admin@<example.com>
 ```
 - `--region` value must remain `"us-east-1"`

2. Verify the domain after receiving verification email, then get
   certificate ARN. From commandline do,

```bash
aws --profile <profile-name> \
    --region <region-name> \
    describe-stacks --stack-name <cert-stack-name>
```

3. Create storage (S3 bucket) and a CDN (cloudfront)

```bash
aws --profile <profile-name> \
    cloudformation create-stack \
    --stack-name <s3cf-stack-name> \
    --region "<insert-AWS-region-here>" \
    --template-body file://./src/s3-cloudfront.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=S3BucketName,ParameterValue="<bucket-name>" \
    ParameterKey=CertificateArn,ParameterValue="<long-cert-ARN-string>"
```
where,
  - `DomainName` is the same as the one defined in the previous step
  - `<bucket-name>` defines a *globally* unique S3 bucket name,
    e.g. `com.example`
  - `<long-cert-ARN-string>` defines the certificate ARN (used for
      cloudfront) obtained in the previous step

3. Wait for the stack creation to finish. Then create a DNS entry
   pointing to cloudfront.

Once the second stack is completed, the cloudfront URL your DNS should
point to is in the output fields:

```bash
aws --profile <profile-name> \
    --region <region-name> \
    describe-stacks --stack-name <s3cf-stack-name>
```

# Long-winded Guide

The deploy is done in a two steps:
1. Create a certificate
2. Create the S3 Bucket and cloudfront, and (optional) Route53 entries

This is because the certificate, as of this writing, *must* be created
in `us-east-1` while the S3 bucket and cloudfront may be created in
any region. Since cloudformation by itself (as of this writing) does
not have a straightforward way of creating resources in two different
regions in the same stack, it's easier to break them up into two
different stacks. These two stacks can of course easily be merged into
a single stack but with the caveat that all the resources would have
to be created in the same region.

0. Prerequisites
 - Unless using a subdomain, must use AWS DNS (Route 53) since normal
   DNS does not allow domain root `CNAME` records.
 - If not already installed, [install](https://aws.amazon.com/cli/)
   `awscli` or activate it.
 - It's assumed that a default aws profile exists. If not, pass
   `--profile <profile-name` to all `aws` commands.
1. Navigate to the root and create the cloudformation stack for
   generating a certificate

```bash
# Note: region must be "us-east-1". Do not change.

aws cloudformation create-stack \
    --stack-name <cert-stack-name> \
    --region "us-east-1" \
    --template-body file://./src/cert.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=VerificationDomain,ParameterValue="<example.com>" \
    [ParameterKey=VerificationMethod,ParameterValue="{EMAIL,DNS}"]
# parameters are defined inside the config yaml file
```

where,
- `VerificationMethod` key-value pair is optional, and can be set
  to either `EMAIL` or `DNS`. If value is set to `EMAIL` (default,
  easier, faster), then AWS will dispatch confirmation emails to
```text
administrator@<example.com>
hostmaster@<example.com>
postmaster@<example.com>
webmaster@<example.com>
admin@<example.com>
```

The domain *must* be confirmed for the `cloudformation` stack
creation to reach completion. Otherwise, if ignored for too
long, the stack creation will timeout, fail, and delete the
certificate.

Alternatively, if you can't get access to those email addresses, set
to `DNS` and enter the required DNS records for your domain and
wait for AWS to verify the entries. You can get the required DNS
records by doing a query on the stack status

```text
aws --region <region-name> \
    describe-stacks --stack-name <cert-stack-name>
```

- `<cert-stack-name>` is the named assigned to the cloudformation
  stack
- `profile-name` is the profile name defined under
  `~/.aws/credentials` and must have appropriate permissions for
  creating Cloudformation stacks, S3 buckets, Cloudfront distribution,
  and Certificate. May be skipped if you have a single unnamed
  profile.
- `region` value [must be](https://docs.aws.amazon.com/acm/latest/userguide/acm-regions.html) `us-east-1` for the cloudfront
  certificate (as of this writing). Do not change.

You can view the status of the stack using AWS console or [using](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/describe-stacks.html)
the aws cli]():

```bash
aws --region <region-name> \
    describe-stacks --stack-name <cert-stack-name>
```

Or perhaps hackingly grep just the ARN info from the stack info
```bash
aws --region <region-name> \
    describe-stacks --stack-name <cert-stack-name> \
    | grep -B1 -C2 "CertArn"
```

2. Create storage (S3 bucket) and a CDN (cloudfront)

```bash
aw  cloudformation create-stack \
    --stack-name <s3cf-stack-name> \
    --region "<insert-AWS-region-here>" \
    --template-body file://./src/s3-cloudfront.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=S3BucketName,ParameterValue="<bucket-name>" \
    ParameterKey=CertificateArn,ParameterValue="<long-cert-ARN-string>" \
    [ParameterKey=CreateRoute53,ParameterValue="{true,false}"
```
where,
    - `DomainName` is the same as the one defined in the previous step
    - `<bucket-name>` defines a *globally* unique S3 bucket name,
      e.g. `com.example`
    - `<long-cert-ARN-string>` defines the certificate ARN (used for
      cloudfront) obtained in the previous step
    - `<insert-AWS-region-here>` is the AWS region where the resources
      are created
    - `<s3cf-stack-name>` is the name assigned to this stack
    - `CreateRoute53` is optional (defaults to `false`) and determines
      whether a Route53 hosted zone along with `A ALIAS` record will
      be created pointing to the cloudfront URL.

This can take a while (e.g. 20 minutes) to complete.

3. Create a DNS entry pointing to cloudfront

If using a subdomain, you can just add a `CNAME` entry to your DNS
configuration list

```text
<website> CNAME <cloudfront-url>
```
where `<cloudfront-url>` can be found from the `CloudFrontURL` output
in the second stack.

If using root domain, must use AWS DNS (Route53) since normal DNS does
not allow domain root `CNAME` records pointing to another URL. With
Route53, you can create an `A` record `ALIAS` pointing to the
`CloudFrontURL`.

4. Website URL that you should redirect in DNS is in the output fields

```bash
aws --region <region-name> \
    describe-stacks --stack-name <cert-stack-name> \
    | grep -B1 -C2 "CertArn"
```

# Undoing Deployment

Delete both stacks:

1. Delete the certificate stack
```bash
aws --region us-east-1 \
    delete-stack --stack-name <cert-stack-name>
```
2. If the S3 bucket is non-empty (see `S3BucketName` above), manually
   delete the S3 bucket

```bash
aws --region <aws-region> \
    s3 rm s3://<S3BucketName> --recursive
```

2. Delete the S3 + Cloudfront stack
```bash
aws --region <stack-region> \
    delete-stack --stack-name <s3cf-stack-name>
```

`<cert-stack-name>` and `<s3cf-stack-name>` are the stack names used
when creating the stacks (see previous section). You can get [list of
stacks](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/list-stacks.html) in a given region using. *Note*: the S3 bucket is
configured to be retain on stack deletion, i.e. the bucket will remain
after stack deletion unless manually removed. Another consequence of
this is that recreating the stack as is will fail unless the bucket is
deleted because the bucket name now already exists.

```bash
aws cloudformation list-stacks
```

# Resources

Official docs of the resources used
- [S3](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html)
- [Certificate Manager](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-certificatemanager-certificate.html)
- [Cloudfront](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-distribution.html)
- [AWS CLI for cloudformation](https://docs.aws.amazon.com/cli/latest/reference/cloudformation)

## Associated AWS pricing

Depending on your use ase, total associated cost can be as low as less
than $1/month:

- [S3](https://aws.amazon.com/s3/pricing/)
- [Certificate Manager](https://aws.amazon.com/s3/pricing/)
- [Cloudfront](https://aws.amazon.com/s3/pricing/)
- [Route 53](https://aws.amazon.com/route53/pricing/)
