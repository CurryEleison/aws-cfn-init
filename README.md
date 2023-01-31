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
constructs. This workaround is a great demonstration of why you want to use CDK instead
of raw CloudFormation (but I use it anyway.)

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

## [Load Balancer, Target Group and traffic control](/cfn/60_alb/)

This one adds an ALB to the mix. The stack is partitioned into three now:
1. One stack for parameters (as in the previous ones)
1. One stack for the ASG and ALB setup
1. One stack for CodeDeploy

The ALB will be unhealthy until the CodeDeploy stack is created and the first
deployment is done, but the stack creates OK. CodeDeploy makes use of both
the autoscaling group and the loadbalancer/targetgroup, so traffic is 
controlled during deployment.

The ALB is on port 80. The required certificate will have to go in a parameter.

I sadly made use of the `StringReflector` lambda in order to be able to have
lists of subnets and security groups as lists in SSM parameters, where the 
SSM parameter name is dynamically generated. Again, this would probably be
better solved by using CDK.

## [Load balancer, target group, deployment combined](cfn/61_alb_combined/)

Nothing new. Just combined CodeDeploy back into the main stack. It's sort of random
if it works or not; the first deployment will frequently fail because the instances
are not ready yet.

## [Certificate on ALB](cfn/62_alb_combined_cert/)

Certificate provisioning from https://github.com/dflook/cloudformation-dns-certificate .
Including the certificate provisioning lambda directly in the stack is according to
recommendation, but I'd probably put it in a separate stack and export the ARN so it
is reusable.

Also adds DNS record for the name in the certificate and redirects HTTP traffic to HTTPS.


## [CodePipeline](cfn/70_codepipeline/)

Adds a CodePipeline that triggers on CloudTrail events. The CloudTrail S3 data 
event was added by hand, but is easy to add in from cfn.

## [CodePipeline triggered by S3 Event](cfn/71_codepipeline_trailevent/)

Same as above, but adds the cloudtrail event. Creates a new bucket for the 
trail since you need a bucket policy -- CloudTrails don't take IAM roles
so they need a resource policy in order to deliver the logs somewhere.