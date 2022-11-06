# Prerequisites

This tutorial will show how to deploy Sigstore on AWS using the AWS Management Console and the AWS CLI.
It is also possible to realize this tutorial using only the CLI or the AWS API.

## AWS

### Install and configure the AWS command line interface

Follow the instructions on the [AWS documentation website](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) to install or update the AWS CLI.
Verify that the `aws` command line tool is installed in version 2:
```
aws
```

If this is your first time using the AWS CLI or if you wish to change an existing configuration, enter `aws configure` and fill up the following fields:
```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-1
Default output format [None]: json
```

### Create a key pair

Create a key pair to make requests to AWS from your local machine:
```
aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem
```

Restrict permissions for the PEM file to be read-only for the user:
```
chmod 400 MyKeyPair.pem
```

For more detailed instructions on set up your AWS remote access, follow the [AWS documentation on how to configure the CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
