# Synopsis

Cloudformation templates for creating infrastructure for static site
hosting with HTTPS support on AWS.

```text
  [Your DNS provider] --> [AWS Route53] --> [AWS Cloudfront] + [Certificate] -> [S3 URL]
```

This assumes that you already own a domain. AWS Route53 DNS may be
skipped altogether depending on your DNS provider.

# Requirements

- [awscli](https://aws.amazon.com/cli/)

# TLDR Quick guide

1. Create certificate stack

```bash
aws --profile <profile-name> \
    cloudformation create-stack \
    --stack-name <cert-stack-name> \
    --region "us-east-1" \   # Do not change!
    --template-body file://./src/cert.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=VerificationDomain,ParameterValue="<example.com>"
```

where
 - `<profile-name>` is the profile name under `~/.aws/credentials`
    with appropriate permissions for creating cloudformation stacks,
    S3 buckets, Cloudfront distribution, and Certificate. This option
    can be skipped there is only one unnamed profile in the config.
 - `<cert-stack-name>` name assigned to the stack
 - `<sub.example.com>` is the domain name e.g. `example.com` or
   `sub.example.com`. Do not use `www.`
 - `--region` value *must* remain `"us-east-1"`
 -  `VerificationDomain`: Root domain where AWS will send emails to
    (`<example.com>` in above example) for verification at
    addresses below

 ```text
 administrator@<example.com>
 hostmaster@<example.com>
 postmaster@<example.com>
 webmaster@<example.com>
 admin@<example.com>
 ```

2. Verify the domain after receiving verification email
3. Get created certificate's ARN (console or cli).

If using the commandline do,

```bash
aws --profile <profile-name> \
    --region "us-east-1" \
    cloudformation describe-stacks --stack-name <cert-stack-name>
```

Example output:

```json
{
 "Stacks": [
    ...
    "Outputs": [
      {
         "Description": "Certificate ARN",
         "OutputKey": "CertArn",
         "OutputValue": "arn:aws:acm:us-east-1:<redacted>:certificate/<redacted>"
     }
   ],
   ...
 ]
}
```

If using the console, either look under the stack's `Outputs` field,
or find the actual certificate and look at its ARN. *Note*: Use the
certificates's ARN, not the stacks!


4. Create storage (S3 bucket) and a CDN (cloudfront)

```bash
aws --profile <profile-name> \
    cloudformation create-stack \
    --stack-name <s3cf-stack-name> \
    --region "<AWS-region>" \
    --template-body file://./src/s3-cloudfront.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=S3BucketName,ParameterValue="<bucket-name>" \
    ParameterKey=CertificateArn,ParameterValue="<long-cert-ARN-string>"
```

where,
  - `DomainName` value (`<sub.example.com>`) is the same as the one
    defined in the previous step
  - `<bucket-name>` defines a *globally* unique S3 bucket name,
    e.g. `com.example`
  - `<long-cert-ARN-string>` defines the certificate ARN (used for
     cloudfront) obtained in the previous step

for more options, see the [long-winded guide](#long-winded-guide)
below.

5. Wait for the stack creation to finish. Then create a DNS entry
   pointing to cloudfront.

Once the second stack is completed, the cloudfront URL your DNS should
point to is in the output fields:

```bash
aws --profile <profile-name> \
    --region <region-name> \
    describe-stacks --stack-name <s3cf-stack-name>
```

Updating your DNS depends on your DNS providers and available options
through them. If using a subdomain, creating a `CNAME` record pointing
to the cloudfront URL is sufficient.

Alternative, if using the root domain, and using Google domains, you
can create couple of entries in your DNS records. See end notes in the
long-winded section.


# Long-winded Guide

The deploy is done in a two steps:
1. Create a certificate
2. Create the S3 Bucket and cloudfront, and (optional) Route53 entries

This is because the certificate, as of this writing, *must* be created
in `us-east-1` while the S3 bucket and cloudfront may be created in
any region. Since cloudformation by itself does not seem to have a
straightforward way of creating resources in two different regions in
the same stack, it is easier to break them up into two different
stacks. These two stacks can of course easily be merged into one but
with the caveat that all the resources would have to be created in the
same region, which is less than ideal.

0. Prerequisites
 - If not already installed, [install](https://aws.amazon.com/cli/) `awscli` or activate it.
 - It's assumed that a default aws profile exists. If not, pass
   `--profile <profile-name` to all `aws` commands.

1. Navigate to the root and create the cloudformation stack for
   generating a certificate.

The profile used must have appropriate permissions for creating
Cloudformation stacks, S3 buckets, Cloudfront distribution, and
Certificate. May be skipped if you have a single unnamed profile.

```bash
# Note: region must be "us-east-1". Do not change.

aws cloudformation create-stack \
    --stack-name <cert-stack-name> \
    --region "us-east-1" \
    --template-body file://./src/cert.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=VerificationDomain,ParameterValue="<example.com>" \
    ParameterKey=VerificationMethod,ParameterValue="{EMAIL,DNS}" # optional
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

The domain *must* be confirmed for the `cloudformation` stack creation
to reach completion. Otherwise, if ignored for too long, the stack
creation will timeout, fail, rollback, and delete the certificate.

Alternatively, if you cannot get access to those email addresses, set
to `DNS` and enter the required DNS records for your domain and wait
for AWS to verify the entries. You can get the required DNS records by
doing a query on the stack status

```text
aws --region <region-name> \
    describe-stacks --stack-name <cert-stack-name>
```
- `<cert-stack-name>` is the named assigned to the cloudformation
  stack
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
aws cloudformation create-stack \
    --stack-name <s3cf-stack-name> \
    --region "<AWS-region>" \
    --template-body file://./src/s3-cloudfront.yaml \
    --parameters ParameterKey=DomainName,ParameterValue="<sub.example.com>" \
    ParameterKey=S3BucketName,ParameterValue="<bucket-name>" \
    ParameterKey=CertificateArn,ParameterValue="<long-cert-ARN-string>" \
    ParameterKey=CreateRoute53,ParameterValue="{true,false}" \ # optional
    ParameterKey=LogCloudfront,ParameterValue="{true, false}"  # optional
```
where,
    - `DomainName` is the same as the one defined in the previous step
    - `<bucket-name>` defines a *globally* unique S3 bucket name,
      e.g. `com.example`
    - `<long-cert-ARN-string>` defines the certificate ARN (used for
      cloudfront) obtained in the previous step
    - `<AWS-region>` is the AWS region where the resources are created
    - `<s3cf-stack-name>` is the name assigned to this stack
    - `CreateRoute53` (optional) can be `true` or `false`
      (default). It is used to specify whether a Route53 hosted zone
      along with `A ALIAS` record will be created (if `true`) pointing
      to the cloudfront URL.
    - `LogCloudfront` (optional) can be `true` or `false`
      (default). If `true`, will create a bucket named
      `<sbucket-name>-accesslog` for cloudfront logs. Note: cloudfront
      console provides *some* logging even if this is set to `false`.

Creation of this second stack may take a while (e.g. 20 minutes) to
complete.

3. Create a DNS entry pointing to cloudfront

Once the last stack is created, get the cloudfront URL using either
the AWS web console or using the `describe-stacks --stack-name
<s3cf-stack-name>` approach highlighted above.

If using a subdomain, you can just add a `CNAME` entry to your DNS
configuration list:

```text
<subdomain> CNAME <cloudfront-url>
www.<subdomain> CNAME <cloudfront-url>
```

Alternative, if using the root domain, and using Google domains, you
can create couple of entries in your DNS records.

- Synthetic record: Subdomain forward `@` to `www.<example.com>`
- `CNAME` record: `www.<example.com> CNAME <cloudfront-url>`

Or, alternatively, if your DNS provider does not support `ALIAS`
records and you are using the root domain for hosting, and you cannot
do the above, you can use AWS DNS (Route53). Refer to `CreateRoute53`
option above.

4. Website URL that you should redirect in DNS is in the output fields

```bash
aws --region <region-name> \
    describe-stacks --stack-name <s3cf-stack-name>
```

# Undoing Deployment

Delete both stacks:

1. Delete the certificate stack
```bash
aws --region us-east-1 \
    delete-stack --stack-name <cert-stack-name>
```
2. If the created S3 bucket (see `S3BucketName` above) is not empty,
   manually delete the S3 bucket

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
configured to be deleted on stack deletion, but the stack cannot
delete the bucket if its contents are not empty. Consequently,
attempting to delete a stack with a non-empty bucket will fail.

# Uploading content

These `s3-cloudfront.yaml` template is hard-coded to have
- `index` as the document root
- `404` as the 404 error page

When uploading these, set the `Content-Type` meta tag to `text/html`
otherwise AWS will likely consider them to be `octet-stream`, and
instead of rendering the content, the browser will likely prompt users
to download the content.

Upload other files (see e.g. `aws s3 sync`) as private, and the
specified bucket policy on S3 will allow them to be served through
Cloudfront.


# Resources

Official docs of the resources used
- [S3](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html)
- [Certificate Manager](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-certificatemanager-certificate.html)
- [Cloudfront](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-distribution.html)
- [AWS CLI for cloudformation](https://docs.aws.amazon.com/cli/latest/reference/cloudformation)

## Associated AWS pricing

Depending on your use case, total associated cost can be as low as
less than $1/month:

- [S3](https://aws.amazon.com/s3/pricing/)
- [Certificate Manager](https://aws.amazon.com/certificate-manager/pricing/)
- [Cloudfront](https://aws.amazon.com/cloudfront/pricing/)
- [Route 53](https://aws.amazon.com/route53/pricing/)
- [AWS description](https://aws.amazon.com/getting-started/projects/host-static-website/services-costs/)
