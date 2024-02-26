# configure-aws-action

A Github Action that configures the AWS SDK to use Github Actions ID Token for authentication
and automatically refreshes the temporary credentials when they expire, with support for multiple profiles and UNIX users.

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
      uses: arianvp/configure-aws-action@main
      with:
        aws-region: us-west-2
        aws-role-arn: arn:aws:iam::123456789012:role/deploy-production
      run: aws s3 cp index.html s3://example-bucket
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
  # FIXME: Probably use a more restrictive policy
  attach_policy_arns = ["arn:aws:iam::aws:policy/AdministratorAccess"]
}


```

[OpenIDConnect]: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-cloud-providers

## Extra AWS Credentials for multiple UNIX users / Daemons

There are scenarios where your Github Workflow might need multiple AWS Credentials.  For example if you have 
a daemon running that is responsible for fetching cached artifacts in the background.  One example of this
is Nix which installs a `nix-daemon` which is responsible for substituting cached artifacts.
Here is an example below that uses two roles to use an S3 bucket as a nix cache. [You can also view the full example here](https://github.com/arianvp/nix-s3-demo)

```yaml
on:
  push:
    branches:
      - main
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    env:
      S3_BUCKET: "nix-s3-demo-1"
      ACCOUNT_ID: "123456123456"
    steps:
      - uses: actions/checkout@v4
      - run: echo "$NIX_PRIVATE_KEY" > /tmp/nix-secret-key
        env:
          NIX_PRIVATE_KEY: ${{ secrets.NIX_PRIVATE_KEY }}
      - uses: DeterminateSystems/nix-installer-action@main
      - name: Set up AWS Credentials for current user
        uses: arianvp/configure-aws-action@main
        with:
          role-arn: arn:aws:iam::${{ env.ACCOUNT_ID }}:role/write-cache
          region: eu-central-1
      - name: Set up AWS credentials for nix-daemon 
        uses: arianvp/configure-aws-action@main
        with:
          role-arn: arn:aws:iam::${{ env.ACCOUNT_ID }}:role/read-cache
          region: eu-central-1
          role-session-name: GithubActionsNixDaemon
          user: root
      # Substitution happens by nix-daemon root user and thus uses the read-cache role
      - run: nix build --extra-trusted-public-keys ${{ secrets.NIX_PUBLIC_KEY }} --extra-substituters 's3://${{ env.S3_BUCKET }}'
      - run: nix store sign --key-file /tmp/nix-secret-key
      # Copying happens by current user and thus uses the write-cache role
      - run: nix copy --to 's3://${{ env.S3_BUCKET }}'
```

## Why use this action instead of `aws-actions/configure-aws-credentials`?

The  `aws-actions/configure-aws-credentials` is a swiss army knife for
configuring AWS credentials. However you really only should be using [OpenIDConnect]
for talking to AWS as it's the most secure way to do so. This action _only_
supports the secure usecase and nothing else.  Furthermore this action
automatically refreshes the temporary credentials when they expire whilst the
`aws-actions/configure-aws-credentials` does not. Also this action supports
muliple profiles and users whilst the upstream action does not.

The `aws-actions/configure-aws-credentials` action calls `sts:AssumeRoleWithWebIdentity`
once and sets environment variables to use the resulting temporary credential.
The problem is in the word **temporary**.  These credentials expire after the default
expiry time of your IAM Role. This means that if you have a step that takes longer
than the expiry time, your action will randomly start failing.

This action does not directly call `sts:AssumeRoleWithWebIdentity` to get
Instead, it configures the AWS SDK by setting the `web_identity_token_file`,
`role_arn`, and `role_session_name` settings in `.aws/config`.  The AWS SDK
will then take care of calling `sts:AssumeRoleWithWebIdentity` to get temporary
credentials when it needs them and will automatically refresh the temporary
credentials when they expire.  This is useful if you have jobs that take longer
than an hour (i.e. the default max role duration time) as the credential will
automatically be renwewed. 

Due to this action not setting environment variables, but writing a config file.
multiple profiles are supported and a single workflow can use multiple AWS IAM
roles in parallel.
