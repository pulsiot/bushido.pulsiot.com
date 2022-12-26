# AWS IAM Roles

## Create IAM user

Lets create a new IAM user and a new IAM Role in AWS Account using AWS CLI. We will require AWS Admin user who have the permission to IAM users, IAM roles and IAM Policies.

Lets create a AWS IAM user for "maurice moss".

```
aws iam create-user --user-name maurice-moss
```

This should give us an output which will contain the ARN of the new IAM user, something like `arn:aws:iam::123456789012:user/maurice-moss` (Replace 123456789012 with your own account.)

## Create IAM policy

Now, lets create IAM Policy

Create a file named "example-policy.json"

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "iam:ListRoles",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    }
  ]
}
```

create "user-define" policy from the above "example-policy.json" file:

```
aws iam create-policy --policy-name example-policy --policy-document file://example-policy.json
```

This should give us an output which will contain the ARN of the new policy, something like `arn:aws:iam::123456789012:policy/example-policy` (Replace 123456789012 with your own account.)

Now, lets attach the "user-define" policy named "example-policy" to the user which we created earlier named "maurice moss"

## Attach IAM policy to a IAM User.

```
aws iam attach-user-policy --user-name maurice-moss --policy-arn "arn:aws:iam::123456789012:policy/example-policy"
```

Also, check to make sure that the attachment is in place using list-attached-user-policies:

```
aws iam list-attached-user-policies --user-name maurice-moss
```

## Create AWS credentials

Now lets create a "access_key_id" and "secret_access_key" for the IAM user "maurice-moss".

```
aws iam create-access-key --user-name maurice-moss
```

Please keep note of these credentials later.

## Create IAM role

Now we will create a IAM role which could be assumed by the user "maurice-moss". Which have read-only access to RDS.

So, lets create a trust relationship policy of the IAM role.

Lets name this file : `example-role-trust-policy.json`

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": { "AWS" : "arn.aws:iam::123456789012:root" },
    "Action": "sts:AssumeRole"
  }
}
```

Lets create a IAM Role with this trust relationship policy.

```
aws iam create-role --role-name example-role --assume-role-policy-document file://example-role-trust-policy.json
```

Now lets attach an "AWS Managed" policy named "arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess" to this role.

```
aws iam attach-role-policy --role-name example-role --policy-arn "arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess"
```

We can verify the managed policies attached to a role using `list-attached-role-policies` :

```
aws iam list-attached-role-policies --role-name example-role
```

## Test IAM user and IAM role

Now we have the IAM user and the IAM role ready, lets confgure AWS CLI for the IAM user "maurice-moss" and verify its permissions.

Lets use `aws configure` or `aws configure --profile=named-profile` command to configure the same. When asked for the Access Key and Secret Key provide the same we collected earlier.

Lets run the below command: to verify the identity of the IAM user.

```
aws sts get-caller-identity
```

You will see the IAM user as `arn:aws:iam::123456789012:user/maurice-moss`

Now try to access the EC2 and RDS instances:

Lets try EC2 as 1st step:

```
aws ec2 describe-instances --query "Reservations[*].Instances[*].[VpcId, InstanceId, ImageId, InstanceType]"
```

This should work flawlessly,

Now, lets try to access RDS as 2nd step:

```
aws rds describe-db-instances --query "DBInstances[*].[DBInstanceIdentifier, DBName, DBInstanceStatus, AvailabilityZone, DBInstanceClass]"
```

This command will generate an access denied error message because IAM user "maurice-moss" doesn't have access to Amazon RDS.

In order for IAM user "maurice-moss" to access RDS instance, he should assume the IAM role named "example-role" as below:

As a 1st step we need to identify the ARN of the IAM role named "example-role"

```
aws iam list-roles --query "Roles[?RoleName == 'example-role'].[RoleName, Arn]"
```

Lets keep not of the ARN from the above command, as we will have to use it with below Assume IAM role command:

```
aws sts assume-role --role-arn "arn:aws:iam::123456789012:role/example-role" --role-session-name AWSCLI-Session
```

Now lets create three environment variables to assume the IAM role.

```
export AWS_ACCESS_KEY_ID=RoleAccessKeyID
export AWS_SECRET_ACCESS_KEY=RoleSecretKey
export AWS_SESSION_TOKEN=RoleSessionToken
```

Now, lets re-try to access RDS using then IAM role:

```
aws rds describe-db-instances --query "DBInstances[*].[DBInstanceIdentifier, DBName, DBInstanceStatus, AvailabilityZone, DBInstanceClass]"
```

## References:

- https://aws.amazon.com/premiumsupport/knowledge-center/iam-assume-role-cli/
- https://www.youtube.com/watch?v=-uogKFE1r60
