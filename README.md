# aws-session

## Introduction

`aws-session` works in concert with the [Amazon AWS CLI](https://aws.amazon.com/cli/) to provide an interactive shell session in which temporary AWS credentials are available to other tools (including the AWS CLI itself). The primary use case for this is accessing AWS services through an IAM account that has [Multi-Factor Authentication](https://aws.amazon.com/iam/details/mfa/) enabled.

In supported shells (`bash`, `zsh`) it also enhances the shell prompt to display the remaining life time of the temporary credentials, and overloads the `aws` command with a shell function that will refresh the credentials first if necessary before delegating to the actual AWS CLI.

## Installation

Before using `aws-session`, you must have the [AWS CLI](https://aws.amazon.com/cli/) installed (through whatever [method](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) you choose) on your PATH and configured (through `aws configure` or manually).

`aws-session` itself is a stand-alone shell script, just download the file and place it somewhere on your PATH, e.g. in `/usr/local/bin`.

## Usage

Simply invoking `aws-session` will run an interactive shell, as will `aws-session shell`. A single command can be executed non-interactively via `aws-session exec COMMAND ...`, e.g. `aws-session exec aws s3 ls`. In either case `aws-session` makes the temporary credentials available in the environment variables

* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`
* `AWS_SESSION_TOKEN` (also copied into `AWS_SECURITY_TOKEN`)

The behavior of `aws-session` can be controlled by a number of command line options or environment variables; generally there is a direct correspondence between the two, and passing a command line option will in fact set the environment variable internally, so that it becomes the default in the `aws-session` sub-shell. Where it makes sense these will be the same environment variables that the AWS CLI and other AWS tools use.

### Credentials and profile

To use a different profile from `~/.aws/credentials`, pass `--profile PROFILE` or set `AWS_DEFAULT_PROFILE`. In either case the profile will passed into the sub-shell environment. Within the sub-shell the profile will generally only affect settings like default region and output format, as the temporary credentials themselves are provided directly via the environment variables listed above.

Note that it is possible to supply the (non-MFA) access key and secret to `aws-session` itself in the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables, though this should rarely be needed. In the sub-shell these will be replaced with the temporary credentials obtained by `aws-session`, but a copy of these "parent" credentials will be stashed in the environment under a different name to enable `aws-session` to obtain new temporary credentials on demand.

### Other options

The desired validity period of the temporary credentials can be specified via the `--session-duration SECONDS` option or the `AWS_SESSION_DURATION` environment variable. The default is to lets AWS itself decide, and depends on the type of credentials being used. For an IAM user, the AWS default is 12 hours. For security reasons, a shorter duration (e.g. 1 or 2 hours) is recommended.

Debug output can be enabled via `--debug` or `AWS_SESSION_DEBUG=1`.

Use of ANSI color codes in the dynamic shell prompt can be enabled or disabled via `AWS_SESSION_COLORS=0/1`, the default is to auto-detect color support via `tput colors`.

## IAM Policy

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
