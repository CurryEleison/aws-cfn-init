# aws-cfn-init
AWS CloudFormation Init on Windows instance

## [Simple Windows](cfn/10_simple_instance/)

Shows `cfn-init` in action on a Windows instance. Mainly, it installs CloudWatch and CodeDeploy agents.

## [Simple Launch Template](cfn/20_asg_launchtemplate/)

Linux instance using `cfn-init` with a launch template and auto-scaling group. Likewise the CloudWatch and CodeDeploy agents are installed.

## [Add IAM](cfn/30_asg_iam/)

Builds on previous example by setting up an instance profile, role etc for the EC2 instances in the auto-scaling group.

## [Playful parameters](cfn/40_asg_parameters/)

Builds further by creating a separate stack that sets up the parameters. The `Namespace` input is used to name the parameters in SSM.
In the stack template we use `{{resolve:ssm:parameter}}` to
figure out the current values of the parameters. This is done so the only necessary input to the stack template is the `Namespace`.

## [CodeDeploy](cfn/50_codedeploy/)

Adds a simple S3 CodeDeploy. Take the files in `app/` zipthem and upload to an S3 bucket. Take the bucket and key
and use them in the parameter stack. 