# cfn-library

Base2Services Common Cloud Formation stacks functionality

## Installation

- As a gem `gem install cfn_manage`
 
- Download source code `git clone https://github.com/base2Services/cfn_manage`

## Functionality

### Stack traversal

Used to traverse through stack and all it's substacks

### Start-stop environment functionality

Stop environment will

- Set all ASG's size to 0
- Stops RDS instances
- If RDS instance is Multi-AZ, it is converted to single-az prior it
  is being stopped
- Disable CloudWatch Alarm actions

Start environment operation will

- Set all ASG's size to what was prior stop operation
- Starts ASG instances
- If ASG instance was Mutli-AZ, it is converted back to Multi-AZ
- Enable CloudWatch Alarm actions
Metadata about environment, such as number of desired/max/min instances within ASG and MultiAZ property
for rds instances, is stored in S3 bucket specified via `--source-bucket` switch or `SOURCE_BUCKET` environment
variable. 

Both start and stop environment operations are idempotent, so if you run `stop-environment`
two times in a row, initial configuration of ASG will persist in S3 bucket (rather than storing 0/0/0) as ASG configuration. 
Same applies for `start` operation - running it against already running environment won't perform any operations. 

In case of some configuration data being lost, script will continue and work with existing data (e.g data about asgs
removed from S3, but rds data persists will results in RDS instances being started)

Order of operations is supported at this point as hardcoded weights per resource type. Pull Requests are welcome 
for supporting dynamic discovery of order of execution - resource tags or local configuration file override are some of
the possible sources. 


## Start - stop cloudformation stack 

### Supported resources

#### AWS::AutoScaling::AutoScalingGroup

**Stop** operation will set desired capacity of ASG to 0

**Start** operation will restore previous capacity

#### AWS::EC2::Instance

**Stop** operation will stop instance [using StopInstances api call](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_StopInstances.html)

**Start** operation will start instance [using StartInstances api call](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_StartInstances.html)


#### AWS::RDS::DBInstance

**Stop** operation will stop rds instance. Aurora is not supported yet on AWS side. Note that RDS instance can be stopped 
for two weeks at maximum. If instance is Multi-AZ, it will get converted to Single-AZ instance, before being stopped
(Amazon does not support stopping Multi-AZ rds instances)


**Start** operation will start rds instance. If instance was running in Multi-AZ mode before being stopped, 
it will get converted to Multi-AZ prior being started

#### AWS::CloudWatch::Alarm

**Stop** operation will disable all of alarm's actions

**Start** operation will enable all of alarm's actions

#### AWS::EC2::SpotFleet

**Stop** operation will set spot fleet target to capacity to 0

**Start** operation will restore spot fleet target to capacity to what was set prior the stack being stopped.

## CLI usage

You'll find usage of `cfn_manage` within `usage.txt` file

```
Usage: cfn_manage [command] [options]

Commands:

cfn_manage stop-environment --stack-name [STACK_NAME]

cfn_manage start-environment --stack-name [STACK_NAME]

cfn_manage stop-asg --asg-name [ASG]

cfn_manage start-asg --asg-name [ASG]

cfn_manage stop-rds --rds-instance-id [RDS_INSTANCE_ID]

cfn_manage start-rds --rds-instance-id [RDS_INSTANCE_ID]


General options

--source-bucket [BUCKET]

    Bucket used to store / pull information from

--aws-role [ROLE_ARN]

    AWS Role to assume when performing start/stop operations.
    Reading and writing to source bucket is not done using this role. 


-r [AWS_REGION], --region [AWS_REGION]

    AWS Region to use when making API calls

-p [AWS_PROFILE], --profile [AWS_PROFILE]

    AWS Shared profile to use when making API calls

--dry-run

    Applicable only to [start|stop-environment] commands. If dry run is enabled
    info about assets being started / stopped will ne only printed to standard output,
    without any action taken.

--continue-on-error

    Applicable only to [start|stop-environment] commands. If there is problem with stopping a resource,
    (e.g. cloudformation stack not being synced or manual resource deletion) script will continue it's 
    operation. By defult script stops when there is problem with starting/stopping resource, and expects
    manual intervention to fix the root cause for failure. 

```

Also, there are some environment variables that control behaviour of the application.
There are command line switch counter parts for all of the

`AWS_ASSUME_ROLE` as env var or `--aws-role` as CLI switch

`AWS_REGION` as env car or `-r`, `--region` as CLI switch

`AWS_PROFILE` as env var or `-p`, `--profile` as CLI switch

`SOURCE_BUCKET` as env var or `--source-bucket` as CLI switch

`DRY_RUN` as env var (set to '1' to enable) or `--dry-run` as CLI switch

## Release process

 - Bump up version `gem install bump && bump [patch|minor|major]`
 - Update timestamp in `cfn_manage.gemspec`
 - Create and publish gem `gem build cfn_manage.gemspec && gem push cfn_manage-$VERSION.gem`
 - Create release page on GitHub 