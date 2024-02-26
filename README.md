# configure-aws-action

A Github Action that configures the AWS SDK to use Github Actions ID Token for authentication
and automatically refreshes the temporary credentials when they expire, with support for multiple profiles and UNIX users.

Use this if you for example have an S3 copy task that takes longer than the max role duration!
Example: Using S3 as a cache for  your nix builds.

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
      values   = ["repo:arianvp/my-repo:environment:production"]
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
Here is an example below that uses two roles to use an S3 bucket as a nix cache. [You can also view and clone the full example here](https://github.com/arianvp/nix-s3-demo)

```yaml
on:
  push:
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    env:
      CACHE_BUCKET: "s3://${{ vars.CACHE_BUCKET }}?region=${{ vars.CACHE_BUCKET_REGION }}"
    steps:
      - uses: actions/checkout@v4
      - run: echo "$NIX_SECRET_KEY" > /tmp/nix-secret-key
        env:
          NIX_SECRET_KEY: ${{ secrets.NIX_SECRET_KEY }}
      - uses: DeterminateSystems/nix-installer-action@main
      - name: Set up AWS Credentials for current user
        uses: arianvp/configure-aws-action@main
        with:
          role-arn: ${{ vars.WRITE_CACHE_ROLE_ARN }}
          region: eu-central-1
      - name: Set up AWS credentials for nix-daemon 
        uses: arianvp/configure-aws-action@main
        with:
          role-arn: ${{ vars.READ_CACHE_ROLE_ARN }}
          region: eu-central-1
          role-session-name: GithubActionsNixDaemon
          user: root
      - run: nix build --extra-trusted-public-keys '${{ secrets.NIX_PUBLIC_KEY }}' --extra-substituters '${{ env.CACHE_BUCKET }}'
      - run: nix store sign --key-file  /tmp/nix-secret-key
      - run: nix copy --to '${{ env.CACHE_BUCKET }}'
```

## Multiple profiles

This action supports setting up multiple profiles. This can be useful for when you have different steps
that require a different role.

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
    steps:
      - uses: actions/checkout@v4
      - uses: arianvp/configure-aws-action@main
        with:
          role-arn: arn:aws:iam::1234561234:role/terraform-plan
          profile: terraform-plan
          region: eu-central-1
      - uses: arianvp/configure-aws-action@main
        with:
          role-arn: arn:aws:iam::1234561234:role/terraform-apply
          profile: terraform-apply
          region: eu-central-1
      - run: AWS_PROFILE=terraform-plan terraform plan
      - run: AWS_PROFILE=terraform-apply terraform apply
```
## Inputs

| Input              | Description                                                                                                                                                        | Required | Default            |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|--------------------|
| `region`           | AWS region.                                                                                                                                                        | No       |                    |
| `role-arn`         | AWS role ARN to assume.                                                                                                                                            | Yes      |                    |
| `role-session-name`| AWS session name.                                                                                                                                                  | No       | `GithubActions`    |
| `profile`          | The AWS profile to write the configuration to. Writes to the default profile if not set                                                                            | No       | `default`          |
| `audience`         | The aud claim in the ID token. Should match a value in client_id_list in the AWS OIDC provider configuration.                                                      | No       | `sts.amazonaws.com`|
| `user`             | Set the AWS configuration for a specific user. If not set, the configuration will be set for the current user. Example: nix-daemon does substitution for nix and needs access to S3. | No       |                    |


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
credentials.
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
