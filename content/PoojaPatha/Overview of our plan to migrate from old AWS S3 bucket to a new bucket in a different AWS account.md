## The Plan

![[Pasted image 20260617152309.png]]

## What to do in the new account?

Create a new bucket in new account with default settings and attach CORS policy:

```json
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "GET",
            "PUT",
            "POST",
            "DELETE"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": []
    },
    {
        "AllowedHeaders": [],
        "AllowedMethods": [
            "GET"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": []
    }
]
```

Then create an user with **IAM User Policy** in the **new account**. (Do **not** put this policy in an S3 bucket policy.).

Setup should be:
1. New Account (444083009123)
2. IAM User: `rails-active-storage`
3. Attach this policy (`PoojaPathaActiveStorageAccess`) to the IAM user:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadOldBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::poojapatha-doc-bucket-prod"
            ]
        },
        {
            "Sid": "ReadOldBucketObjects",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::poojapatha-doc-bucket-prod/*"
            ]
        },
        {
            "Sid": "ListNewBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::poojapatha-production-storage"
            ]
        },
        {
            "Sid": "ManageNewBucketObjects",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:AbortMultipartUpload"
            ],
            "Resource": [
                "arn:aws:s3:::poojapatha-production-storage/*"
            ]
        }
    ]
}
```

![](attachments/Pasted%20image%2020260617144809.png)

## What needs to be changed in the old account?

Add the following bucket policy to the old s3 bucket with the ARN of the new IAM user of the new AWS account. So that the new user can read from old bucket.

1. Bucket: `poojapatha-doc-bucket-prod`
2. Add a **Bucket Policy**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowRailsMigrationUserListBucket",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::444083009123:user/rails-active-storage"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::poojapatha-doc-bucket-prod"
        },
        {
            "Sid": "AllowRailsMigrationUserReadObjects",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::444083009123:user/rails-active-storage"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::poojapatha-doc-bucket-prod/*"
        }
    ]
}
```

## Test

Run the following using the new account credentials:

```shell
AWS_ACCESS_KEY_ID=<new S3 user> \    
AWS_SECRET_ACCESS_KEY=QwgFNA6u2ixtpWTsiVRpWF3UV9F6gY7gLp/P6/MG \
aws sts get-caller-identity
```

Expected:

```json
{
  "Account": "444083009123",
  "Arn": "arn:aws:iam::444083009123:user/rails-active-storage"
}
```

Now check if the new user is able to access the old S3 bucket using new credentails:

```shell
AWS_ACCESS_KEY_ID=<new-account-key> \
AWS_SECRET_ACCESS_KEY=<new-account-secret> \
aws s3 ls s3://poojapatha-doc-bucket-prod
```

This should list the objects in the old s3 using new s3 user credentials:

```shell
2026-05-06 11:55:34    1160354 0020xd7dz2igm80bwia0tewyplpb
2026-05-07 09:43:51     641201 002qccs6srgrrkmbbh8mf6329ux9
2026-04-28 11:15:33     115839 004yv3l84qv37i16kggigrycgnha
...
...
```

Now we can proceed to use our migration script:

```shell
# Dry-run (default) — see what would be renamed, get the CSV locally
bin/migrate_s3 --normalize

# Live rename
bin/migrate_s3 --normalize --no-dry-run

# On staging
bin/migrate_s3 --normalize --no-dry-run --env=staging
```

Thanks!