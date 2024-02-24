# configure-aws
A Github Action that configures the AWS SDK to use Github Actions ID Token for authentication

## Why use this action instead of `aws-actions/configure-aws-credentials`?

The  `aws-actions/configure-aws-credentials` is a swiss army knife for
configuring AWS credentials. However you really only should be using [OpenIDConnect][OpenID Connect]
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

Please refer to the documentation in [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
for setting up your AWS account to use OpenID Connect with Github Actions. You can use this action
as a drop-in replacement for `aws-actions/configure-aws-credentials` in the examples in that documentation.

A minimal example of using this action is:
    
```yaml
name: Deploy to AWS
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      id-token: write
    steps:
    - name: Configure AWS
      uses: aws-actions/configure-aws@main
      with:
        aws-region: us-west-2
        aws-role-arn: arn:aws:iam::123456789012:role/deploy-production
      run: aws sts get-caller-identity
```
A minimal example of configuring AWS to use OpenID Connect with Github Actions is:

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["ffffffffffffffffffffffffffffffff"]
}

data "aws_iam_policy_document" "assume_github" {
  statement {
    actions   = ["sts:AssumeRoleWithWebIdentity"]
    effect    = "Allow"
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:arianvp/configure-aws:environment:production"]
    }
  }
}

resource "aws_iam_role" "github" {
  name               = "deploy-production"
  assume_role_policy = data.aws_iam_policy_document.assume_github.json
  # TODO: use a more restrictive policy
  attach_policy_arns = ["arn:aws:iam::aws:policy/AdministratorAccess"]
}


```

[OpenIDConnect]: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-cloud-providers