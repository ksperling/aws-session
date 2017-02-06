# aws-session

[![Build Status](https://travis-ci.org/ksperling/aws-session.svg?branch=master)](https://travis-ci.org/ksperling/aws-session)
[![Code Climate](https://codeclimate.com/github/ksperling/aws-session/badges/gpa.svg)](https://codeclimate.com/github/ksperling/aws-session)

## Overview

`aws-session` works in concert with the [Amazon AWS CLI](https://aws.amazon.com/cli/) to provide an interactive shell session in which temporary AWS credentials are available to other tools (including the AWS CLI itself). The primary use case for this is accessing AWS services through an IAM account that has [Multi-Factor Authentication](https://aws.amazon.com/iam/details/mfa/) enabled.

In supported shells (`bash`, `zsh`) it also enhances the shell prompt to display the remaining life time of the temporary credentials, and overloads the `aws` command with a shell function that will refresh the credentials first if necessary before delegating to the actual AWS CLI. A refresh can also be triggered manually via the shell function `aws-session-refresh`.

## Installation

Before using `aws-session`, you must have the [AWS CLI](https://aws.amazon.com/cli/) installed (through whatever [method](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) you choose) on your PATH and configured (through `aws configure` or manually).

`aws-session` itself is a stand-alone shell script, just [download the script](https://raw.githubusercontent.com/ksperling/aws-session/master/aws-session) and place it somewhere on your PATH, e.g. in `/usr/local/bin`.

## Usage

Simply invoking `aws-session` will run an interactive shell, as will `aws-session shell`. A single command can be executed non-interactively via `aws-session exec COMMAND ...`, e.g. `aws-session exec aws s3 ls`. In either case `aws-session` makes the temporary credentials available in the environment variables

* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`
* `AWS_SESSION_TOKEN` (also copied into `AWS_SECURITY_TOKEN`)

The behavior of `aws-session` can be controlled by a number of command line options or environment variables; generally there is a direct correspondence between the two, and passing a command line option will in fact set the environment variable internally, so that it becomes the default in the `aws-session` sub-shell. Where it makes sense these will be the same environment variables that the AWS CLI and other AWS tools use.

### Credentials and profile

To use a different profile from `~/.aws/credentials`, pass `--profile PROFILE` or set `AWS_DEFAULT_PROFILE`. In either case the profile will be passed into the sub-shell environment. Within the sub-shell the profile will generally only affect settings like the region and output format, as the temporary credentials themselves are provided directly via the environment variables listed above.

Note that it is possible to supply the (non-MFA) access key and secret to `aws-session` itself in the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables, though this should rarely be needed. In the sub-shell these will be replaced with the temporary credentials obtained by `aws-session`, but a copy of the "parent" credentials will be stashed in the environment under a different name to enable `aws-session` to obtain new temporary credentials on demand.

### MFA Activation

Using the command `aws-session provision-mfa` a virtual MFA device can be created and activated from the command line. The QR Code used to seed the MFA device is rendered in the console itself. This is supported "out of the box" on OS X; on other platforms `convert` / ImageMagick must be installed. A [sample IAM Policy](#IAM-Policies) to allow users to provision their own MFA device via this command is provided below.

### Other options

The desired validity period of the temporary credentials can be specified via the `--session-duration SECONDS` option or the `AWS_SESSION_DURATION` environment variable. The default is to lets AWS itself decide, and depends on the type of credentials being used. For an IAM user, the AWS default is 12 hours. For security reasons, a shorter duration (e.g. 1 or 2 hours) is recommended.

Debug output can be enabled via `--debug` or `AWS_SESSION_DEBUG=1`.

Use of ANSI color codes in the dynamic shell prompt can be enabled or disabled via `AWS_SESSION_COLORS=0/1`, the default is to auto-detect color support via `tput colors`.

## IAM Policies

The following IAM policy represents the minimal permissions required for a user to use temporary STS credentials via `aws-session`:

```
{ "Version": "2012-10-17",
  "Statement": [ {
    "Action": "sts:GetCallerIdentity",
    "Resource": "*",
    "Effect": "Allow"
  }, {
    "Action": "iam:ListMfaDevices",
    "Resource": "arn:aws:iam::*:user/${aws:username}",
    "Effect": "Allow"
  }, {
    "Action": "sts:GetSessionToken",
    "Resource": "*",
    "Condition": { "Bool": { "aws:SecureTransport": "true" } },
    "Effect": "Allow"
  } ]
}
```

The following policy can be used to allow users to create a virtual MFA device for themselves using `aws-session provision-mfa`. Note that AWS itself prevents an MFA device from being deleted without deactivating it first. This policy intentionally only allows deletion but not deactivation.

```
{ "Version": "2012-10-17",
  "Statement": [ {
    "Action": "sts:GetCallerIdentity",
    "Resource": "*",
    "Effect": "Allow"
  }, {
    "Action": [ "iam:ListMfaDevices", "iam:CreateVirtualMfaDevice", "iam:DeleteVirtualMfaDevice", "iam:EnableMfaDevice" ],
    "Resource": [ "arn:aws:iam::*:user/${aws:username}", "arn:aws:iam::*:mfa/${aws:username}" ],
    "Condition": { "Bool": { "aws:SecureTransport": "true" } },
    "Effect": "Allow"
  } ]
}
```
