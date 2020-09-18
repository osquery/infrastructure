# Migration

This is a short recap of migrating AWS infrastructure from a Facebook-managed account to a TSC-managed account.
This recap follows the order in which infrastructure was migrated.

## ACM

Using the `osquery-infra` account:

This is a straightforward process within the ACM tool using the web interface.
Request a `*.osquery.io` certificate and use DNS verification.

Then in the Facebook-managed account, which still hosts the DNS zone, set the verification CNAME record.

Now the TSC-managed account has a TLS certificate to use.

## CloudFront

Using the `osquery-infra` account:

Set up a default distribution for the origin `osquery.github.io/osquery-site`, and set `osquery.io` and `www.osquery.io` as the CNAMEs.
Use the ACM certificate from before.

Set up a second distribution for the S3 origin `osquery-packages.s3.amazonaws.com`, and set `pkg.osquery.io` as the CNAME.
Use the ACM certificate from before.

Then in the Facebook-managed account, which still hosts the DNS zone, set the A/AAAA records to the new distribution IDs in the TSC-managed account.

## Route53

Using the `osquery-infra` account:

Manually create the 8 records within the `osquery.io` zone:

- `osquery.io` (A and AAAA) to the new CloudFront distribution ID.
- `www.osquery.io` (A and AAAA) to the new CloudFront distribution ID.
- `pkg.osquery.io` (A and AAAA) to the new CloudFront distribution ID.
- `p.osquery.io` (PTR) to `pkg.osquery.io`, a legacy record, consider removing.
- And the gSuite MX records.

We do not control the DNS registration so email the Linux Foundation asking them to update the NS record to point to the new zone.

## S3 ('clone' bucket)

Using the Facebook-managed account:

Set up a bucket policy similar to the following on `osquery-packages`:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DelegateS3Access",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::834249036484:user/theopolis-identity"
            },
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:GetObjectTagging",
                "s3:GetObjectAcl",
                "s3:GetObjectVersionAcl",
                "s3:GetBucketVersioning",
                "s3:ListBucketMultipartUploads",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::osquery-packages/*",
                "arn:aws:s3:::osquery-packages"
            ]
        }
    ]
}
```

This allows the TSC-managed account to eventually use `sync` to copy the content.

Using the `osquery-storage` account:

Create a bucket called `osquery-packages-xfer`, this will keep a copy of `osquery-packages`.
NB: This new bucket was created in `us-east-2` as an oversight. The original bucket was in `us-east-1` and it is MUCH faster to copy/sync/etc in the same region.

Use the following policy on `osquery-packages-xfer`:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::osquery-packages-xfer/*"
        },
        {
            "Sid": "DelegateS3Access",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::834249036484:user/theopolis-identity"
            },
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:PutObject",
                "s3:GetObjectTagging",
                "s3:PutObjectTagging",
                "s3:GetObjectAcl",
                "s3:PutObjectAcl",
                "s3:GetObjectVersionAcl",
                "s3:PutObjectVersionAcl",
                "s3:ListBucketMultipartUploads",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::osquery-packages-xfer/*",
                "arn:aws:s3:::osquery-packages-xfer"
            ]
        }
    ]
}
```

This includes "get" and "put" because we'll need to perform more actions after the sync.

Run the sync:

```
aws s3 sync s3://osquery-packages s3://osquery-packages-xfer --source-region us-east-1 --region us-east-2 --debug
```

Once this is finished we want to "smoke test" that this works. Create a new distribution in CloudFront for this S3 bucket and call it `pkg-xfer.osquery.io`, and try to download something.
NB: This originally failed! Here what was needed to fix.

```
aws s3 cp --recursive --acl bucket-owner-full-control s3://osquery-packages-xfer s3://osquery-packages-xfer --storage-class STANDARD
```

Then in the web console, select all of the folders and use the "Make Public" action.

Delete the "xfer" CloudFront distribution and the `pkg-xfer.osquery.io` records as a cleanup.

## S3 ('restore' bucket)

Now the TSC-managed `osquery-storage` account has a clone of the `osquery-packages` content.

The next step is to delete the `osquery-packages` bucket, wait an hour for AWS's mandatory 'cool down' and recreate in the TSC-managed account.

**First bug!** This took over 5 hours since there was a `osquery-logs` versioned folder that had 50G of 4k files, enumerating and deleting 10k files at a time took a while. I tried a few of the following: https://gist.github.com/weavenet/f40b09847ac17dd99d16

What worked was:

```
import boto3
session = boto3.session()
s3 = session.resource(service_name='s3')
bucket = s3.Bucket('osquery-packages')
bucket.object_versions.delete()
```

What could have gone better here was first deleting the `osquery-logs` folder, then deleting everything else. The `osquery-logs` folder was deleted last as it was already deleted and only existed as a tombstone.

After this, you can follow the same steps from `osquery-packages` to `osquery-packages-xfer` but in reverse.

**Second bug!** When creating the new bucket we used `us-east-2`, where it was originally hosted in `us-east-1`.
You can see that this eventually creates an error https://github.com/osquery/osquery/issues/6653 if folks use `https://s3.amazonaws.com/osquery-packages` instead of the S3-recommended `https://osquery-packages.s3.amazonaws.com`.

Instead of having everyone update their repo files, we deleted the `osquery-packages` bucket yet again and re-created using `us-east-2`.

What could have gone better here was sticking with the `us-east-1` region.
