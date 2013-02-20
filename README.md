rake_stack
==========

Rake tasks for AWS CloudFormation templates and stacks. 

Clone the project, modify the template, run, and re-run, the Rake tasks.

Features:

* Validate the template with CloudFormation directly
* Merge a common Mappings section into the template
* Merge resource specific Cloud-Init UserData into the template
* Extract template Parameters section to a YAML file
* Configuration, Parameters, Outputs, Ids in YAML files
* Get stack events to stdout
* Create an Amazon S3 bucket with IAM Policy for Cost Allocation and Billing Reports
* Create, Update, Delete, a CloudFormation stack
* Create Amazon S3 buckets with necessary policy for billing and cost allocation reports
* Download billing and cost allocation reports from the S3 bucket
* Create an Amazon S3 bucket with public access policy for stagin the CloudFormation template

Config:

Add your AWS access keys.

``` bash
vi config.yml
```

Edit UserData shell script. This example will be merged into the UserData property of the Ec2Instance resource

``` bash
vi userdata_Ec2Instance.sh
```

Example Usage:

Validate and create a CloudFormation Stack:

``` bash
rake merge
vi template.json
rake validate
rake parameters
vi parameters.yml
rake create
```

Stage to Amazon S3:

```bash
rake validate
rake buckets
rake stage
```
