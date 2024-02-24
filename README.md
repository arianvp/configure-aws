# configure-aws
A Github Action that configures the AWS SDK to use Github Actions ID Token for authentication

## Why use this action instead of `aws-actions/configure-aws-credentials`?

The  `aws-actions/configure-aws-credentials` is a swiss army knife for
configuring AWS credentials. However you really only should be using [OpenID
Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-cloud-providers)
for talking to AWS as it's the most secure way to do so. This action _only_
supports the secure usecase and nothing else.  Furthermore this action
automatically refreshes the temporary credentials when they expire whilst the
`aws-actions/configure-aws-credentials` does not.

The `aws-actions/configure-aws-credentials` action calls `sts:AssumeRoleWithWebIdentity`
once and sets environment variables to use the resulting temporary credential.
The problem is in the word **temporary**.  These credentials expire after the default
expiry time of your IAM Role. This means that if you have a step that takes longer
than the expiry time, your action will randomly start failing.

This action does not directly call `sts:AssumeRoleWithWebIdentity` to get
Instead, it configures the AWS SDK by setting the `AWS_WEB_IDENTITY_TOKEN_FILE`,
`AWS_ROLE_ARN`, and `AWS_ROLE_SESSION_NAME` environment variables.  The AWS SDK
will then take care of calling `sts:AssumeRoleWithWebIdentity` to get temporary
credentials when it needs them and will automatically refresh the temporary
credentials when they expire.

## Usage
    
```yaml
name: Deploy to AWS
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
    - name: Configure AWS
      uses: aws-actions/configure-aws@main
      with:
        aws-region: us-west-2
        aws-role-arn: arn:aws:iam::123456789012:role/github-actions
      run: aws sts get-caller-identity
```
