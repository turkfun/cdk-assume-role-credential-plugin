# CDK Assume Role Credential Plugin

This is a CDK credential plugin that assumes a specified role
in the Stack account.

## When would I use this plugin

There are two main use cases that this plugin addresses.

1. You have a CDK application that deploys stacks to multiple AWS accounts.
2. You have a CDK application that deploys a stack to an AWS account that is different than the current AWS account.

## What it does

This plugin allows the CDK CLI to automatically obtain AWS credentials from a stack's target AWS account.
This means that you can run a single command (i.e. cdk synth) with a set of AWS credentials, and the CLI will
determine the target AWS account for each stack and automatically obtain temporary credentials for the target
AWS account by assuming a role in the account.

For more details on the credential process see the [How does it work](#how-does-it-work) section below.

## Prerequisites

In order to use the plugin in a CDK app you have to first perform a couple prerequisites

### Install the plugin

This is not currently published to npm, so you will need to install it directly from github instead.

If you are running the CDK cli from a global install you'll need to install the
plugin globally as well.

```bash
$ npm install -g git+https://github.com/aws-samples/cdk-assume-role-credential-plugin.git
```

If you are running from a locally installed version of the CDK cli (i.e. `npm run cdk` or `npx cdk`) you can
install the plugin locally

```bash
$ npm install git+https://github.com/aws-samples/cdk-assume-role-credential-plugin.git
```

I would recommend installing the plugin both locally and globally so that the plugin can be used both on a
development machine as well as part of a CI/CD pipeline.

You must then tell the CDK app to use the plugin. This can be done in two ways.

1. cdk.json file

`example cdk.json`
```json
{
  "app": "npx ts-node bin/my-app",
  "plugin": ["cdk-assume-role-credential-plugin"]
}
```

2. via the `--plugin` option on the cli

```bash
$ cdk synth --plugin cdk-assume-role-credential-plugin
```

### Set context values (optional)

This plugin needs to know the name of the IAM roles to assume in the target AWS account. By default
it looks for IAM roles with the names:

- `cdk-readOnlyRole` (for read only operations)
- `cdk-writeRole` (for write operations)

If you would like to privide your own custom values you can do so through setting context values.

The plugin will look for two context keys, which can either be set in the `cdk.context.json`
file or via the `--context` option on the cli.

- `assume-role-credentials:readIamRoleName`: The role name of the role in the stack account that will be assumed to perform read only
activities.
- `assume-role-credentials:writeIamRoleName`: The role name of the role in the stack account that will be assumed to perform write activities.

`example cdk.context.json`
```json
{
  "assume-role-credentials:writeIamRoleName": "writeRole",
  "assume-role-credentials:readIamRoleName": "readRole"
}
```

`example cli`
```bash
$ cdk synth --context assume-role-credentials:writeIamRoleName=writeRole --context assume-role-credentials:readIamRoleName=readRole
```

## Using the plugin

Once the [prerequisites](#prerequisites) are completed the CDK CLI will automaticlly attempt to use the credential plugin
if the default credentials do not work for the stack's target AWS account.

For example, suppose I had a CDK application that deployed 2 stacks, each to a different AWS account.
I am deploying these stacks from a 3rd AWS, so the CDK CLI will automatically attempt to use the plugin
to obtain credentials for the target accounts.

My CDK app
```typescript
const dev  = { account: '2383838383', region: 'us-east-2' };
const prod = { account: '8373873873', region: 'us-east-2' };

new MyAppStack(app, 'dev', { env: dev });
new MyAppStack(app, 'prod', { env: prod });
```

I'll run a single command to synthesize the application.

```bash
$ cdk synth

Successfully synthesized to /myapp/cdk.out
Supply a stack id (dev, prod) to display its template.
```

If you want to see what is happening behind the scenes you can run the command with verbose logging enabled.

*removing logs that aren't related to the plugin*
```bash
$ cdk synth -v

...
AssumeRoleCredentialPlugin found value for readIamRole cdk-readOnlyRole. checking if we can obtain credentials
AssumeRoleCredentialPlugin found value for writeIamRole cdk-writeRole. checking if we can obtain credentials
canProvideCredentails for read role: true
canProvideCredentails for write role: true
Using AssumeRoleCredentialPlugin credentials for account 2383838383
AssumeRoleCredentialPlugin getting credentials for role arn:aws:iam::2383838383:role/cdk-readOnlyRole with mode 0
...

Successfully synthesized to /myapp/cdk.out
Supply a stack id (dev, prod) to display its template.
```

## How does it work

The CDK has the concept of [environments](https://docs.aws.amazon.com/cdk/latest/guide/environments.html)
An environment is a combination of the target AWS account and AWS region into which an individual stack is
intended to be deployed.

A CDK app can contain multiple stacks the are deployed to multiple environments. A simple example of this
would be an application that deployed a `dev` stack into a `dev` AWS account and a `prod` stack into a `prod`
AWS account. This would look something like:

```typescript
const dev  = { account: '2383838383', region: 'us-east-2' };
const prod = { account: '8373873873', region: 'us-east-2' };

new MyAppStack(app, 'dev', { env: dev });
new MyAppStack(app, 'prod', { env: prod });
```

When you run a cdk command such as `synth` or `deploy` the cli will need to perform actions against the AWS
account that is defined for the stack. It will attempt to use your default credentials, but what happens if you
need credentials for multiple accounts? This is where credential plugins come into play. The basic flow that the
cli will take when obtaining credentials is:

1. Determine the `environment` for stack
2. Look for credentials that can be used against that environment.
  1. If it can find credentials in the DefaultCredentialChain then it will use those.
  2. If it can't find any, then it will load any credential plugins and attempt to fetch credentials for the
  environment using the credential plugins

Without using a credential plugin you would need to manually obtain credentials for each environment and then
run the cli for that stack. A common script would be something like this:

```bash
#!/bin/bash

ASSUME_ROLE_ARN=$1
SESSION_NAME=$2
STACK=$3

creds=$(mktemp -d)/creds.json
echo "assuming role ${ASSUME_ROLE_ARN} with session-name ${SESSION_NAME}"
aws sts assume-role --role-arn $ASSUME_ROLE_ARN --role-session-name $SESSION_NAME > $creds
export AWS_ACCESS_KEY_ID=$(cat ${creds} | grep "AccessKeyId" | cut -d '"' -f 4)
export AWS_SECRET_ACCESS_KEY=$(cat ${creds} | grep "SecretAccessKey" | cut -d '"' -f 4)
export AWS_SESSION_TOKEN=$(cat ${creds} | grep "SessionToken" | cut -d '"' -f 4)

npm run cdk synth -- $STACK -o dist
```

Which you would then exexute for each stack:
```bash
$ ./assume_role_script.sh arn:aws:iam::2383838383:role/synthRole synth dev
$ ./assume_role_script.sh arn:aws:iam::8373873873:role/synthRole synth prod
```

This can become difficult to maintain, especially across multiple projects and with a CI/CD pipeline. Instead
you can just install a credential plugin and execute a single command and the cli will obtain the appropriate
credentials for each stack.

```bash
$ cdk synth
getting credentials for role arn:aws:iam::2383838383:role/synthRole with mode 0
getting credentials for role arn:aws:iam::8373873873:role/synthRole with mode 0
Successfully synthesized to cdk.out
Supply a stack id (dev, prod) to display its template.
```

