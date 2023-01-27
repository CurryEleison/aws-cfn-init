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

The CodeDeploy app tries to do a deploy as soon as the deployment group is created, but this will fail as the instance isn't
ready yet. You can work around this by commenting out the CodeDeploy resources and creating the stack. Then wait a bit, comment
in the CodeDeploy resources again and update the stack. I'll think about making a custom resource that waits for everything to 
be ready and then make the deployment group dependent on that.

## [String Reflector](cfn/51_parameter_customresource/)

Demonstrates how to work around issues with resolution order between `!Split` `!Sub` and `{{resolve:...}}`
constructs. 

Cloudformation will always resolve the `!Fn::*` first and do the `{{resolve:...}}` later. So they are tricky
to combine. E.g. this

```
!Split [",", !Sub "{{resolve:ssm:/${Namespace}/SecurityGroups}}"]
```

will correctly retrieve the parameter from SSM parameter store, but the `!Split` will happen _before_
the parameter is retrieved, so the contents of the parameter won't be split. To work around this we
set up a CFN Custom Resource, with a backing lambda that simply reflects back the properties of the
resource as attributes. Thus you can go

```yaml
Parameters:
  Namespace:
    Type: String
    Default: Development
Resources:
  StringReflectorResourceBackend:
    Type: "AWS::Lambda::Function"
    Properties:
      ...
  StringReflector:
    Type: "AWS::CloudFormation::CustomResource"
    Properties:
      ServiceToken: !GetAtt StringReflectorResourceBackend.Arn
      SecurityGroups: !Sub "{{resolve:ssm:/${Namespace}/SecurityGroups}}"
  SimpleLaunchTemplate:
    Type: "AWS::EC2::LaunchTemplate"
    Properties:
      LaunchTemplateData:
        SecurityGroupIds: !Split [",", !GetAtt StringReflector.SecurityGroups]
```
