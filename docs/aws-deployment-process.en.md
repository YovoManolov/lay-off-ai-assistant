# AWS infrastructure deployment process

*Bulgarian version: [`aws-deployment-process.md`](aws-deployment-process.md).*

Split by who does what: Yana, who needs no technical knowledge, and the
engineer, once they have access. Phases 0 and 2 are the engineer's; Phase 1 is
hers.

## Phase 0 - engineer's preparation (before anything is sent to Yana)

1. **Write a CloudFormation template** that creates:
   - an IAM user for the engineer
   - an IAM policy scoped to the required services only (Lambda, API Gateway,
     Amplify, Bedrock, CloudWatch Logs)
   - an explicit `Deny` on billing and account-management actions
   - an AWS Budgets budget with a notification that emails Yana's address when
     spend crosses a given threshold
   - IAM roles for the application itself, under fixed names - a Lambda
     execution role (`lay-off-assistant-lambda-exec`) and an Amplify service
     role (`lay-off-assistant-amplify`) - plus `iam:PassRole` on their ARNs in
     the engineer's policy. The engineer can then pass the roles without being
     able to create new ones.
   - `iam:CreateAccessKey` and `iam:DeleteAccessKey`, scoped to the engineer's
     own user ARN. That is enough for them to issue programmatic credentials
     for Terraform, without seeing or touching any other principal.
   - optionally, an S3 bucket for Terraform state, with `s3:*` on that one
     bucket and nothing else.

   The last three items are only needed if the deployment will go through
   Terraform (see Phase 2). For a manual console deployment the template can
   stop at the budget notification.

2. **Publish the template** somewhere reachable (an S3 bucket with public read
   access is the simplest option), or just send the `.yaml` file to Yana
   directly.
3. **Generate a "Launch Stack" quick-create URL** in the form:
   ```
   https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=<TEMPLATE_URL>&stackName=lay-off-assistant-infra
   ```

## Phase 1 - what Yana does (no technical knowledge required)

1. **Create a new AWS account** under her email, with her card. This is the
   only step where her personal and payment details enter the process at all -
   they never reach the engineer.
2. **Open the link** the engineer sends. It opens the CloudFormation console
   with the template already filled in.
3. **Choose "Create stack"** and wait (usually under a minute) for the status
   to reach `CREATE_COMPLETE`.
4. **Open the stack's "Outputs" tab** - three values are waiting there:
   - the IAM sign-in URL
   - the user name
   - the temporary password
5. **Send those three values to the engineer** (over email or Slack, say).
   Her involvement ends here.

## Phase 2 - what the engineer does

1. **Sign in** with the IAM sign-in URL, the user name and the temporary
   password.
2. **Change the password** on first sign-in (AWS forces this for a new IAM user
   created with a temporary password).
3. **Check or request access to the Bedrock models** in the console
   (Bedrock → Model access) - the Anthropic models must be explicitly enabled
   once per Region before they can be invoked through the API.
4. **Deploy the frontend on AWS Amplify** - connect the Git repository holding
   the chat interface, or upload the static files directly.
5. **Deploy the backend**:
   - Lambda function(s) carrying the routing and orchestration logic (the
     subagents' markdown files are used as system prompts)
   - an API Gateway route pointing at the Lambda

   Steps 4 and 5 can also be done with Terraform - see "Terraform instead of a
   manual deployment" below.

6. **Test end-to-end** - send a test message through the UI and confirm it
   reaches Bedrock and comes back with a response.
7. **Verify the budget notification** - confirm the threshold and Yana's email
   address are set correctly. The template already created it in Phase 0; this
   step only checks it.
8. **Optional: a domain** - if a custom domain is preferred over the default
   Amplify subdomain, register it through Route 53 and attach it to the
   Amplify app.

### Terraform instead of a manual deployment

Steps 4 and 5 do not have to be manual. Terraform talks to the same AWS APIs
and does not care whether a resource was created from the console, by
CloudFormation, or by Terraform itself. **Phase 1 does not change** - Yana
still clicks the link and sends the same three values; the whole difference
sits in the Phase 0 template.

1. **Issue an access key** - IAM → Users → the engineer's user → Security
   credentials → Create access key. The Phase 1 password only opens the
   console, and Terraform needs a key pair.
2. **Read the account ID out of the sign-in URL** - it has the form
   `https://<account-id>.signin.aws.amazon.com/console`. The account ID plus
   the role names fixed in the template gives both role ARNs. Nothing further
   has to be asked of Yana.
3. **Decide where state lives** - the Phase 0 S3 bucket (with native state
   locking) or a local file. Local state is fine for a one-off deployment by a
   single operator, but it stays on that operator's machine.
4. **Write and apply the configuration** for the Lambda functions, the API
   Gateway route, the Amplify app and the CloudWatch Logs log groups, pointing
   `role_arn` / service role at the roles that already exist rather than
   creating new ones.

What stays outside Terraform:

- **The Phase 0 resources** - the IAM user, the policy and the budget belong to
  the CloudFormation stack. No `terraform import` on any of them: a single
  `terraform destroy` would take out the engineer's own access and Yana's spend
  alert.
- **Bedrock model access** (step 3) - enabled once per Region from the console,
  and it stays a manual step.

## Why the process is split this way

Yana never creates IAM users, never writes policies, and never sees anything to
do with permissions - she clicks a link and copies three lines of text. The
engineer never sees the payment method or the account's root credentials - they
work entirely through a restricted IAM user that the template created for them.

Whether they deploy by hand from the console or with Terraform is entirely
their call, and it is invisible from Yana's side - she walks through the same
five steps either way.
