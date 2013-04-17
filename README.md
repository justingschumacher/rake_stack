rake_stack
==========

Rake tasks for AWS CloudFormation templates and stacks. 

Clone the project, modify the template, run, and re-run, the Rake tasks.

# Features:

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
* Create an Amazon S3 bucket with public access policy for staging the CloudFormation template
* Sum total allocated cost of stack run (tax and credits are applied) from cost allocation reports.

# Config:

## Add your AWS access keys.

``` bash
vi config.yml
```

## Edit UserData shell script. This example will be merged into the UserData property of the Ec2Instance resource

``` bash
vi userdata_Ec2Instance.sh
```

# Example Usage:

## Validate and create a CloudFormation Stack:

``` bash
rake merge
vi template.json
rake validate
rake parameters
vi parameters.yml
rake create
```

## Stage to Amazon S3:

```bash
rake validate
rake buckets
rake stage
```

The stage task includes glob patterns in staging_files_manifest via [FileList](http://rake.rubyforge.org/classes/Rake/FileList.html "Class::Rake::FileList"), and excludes what is in .gitignore.

## Enable collection of reports and get cost:

```bash
rake buckets
rake reports
rake cost
```

## All Tasks

```bash
✗ rake -T
rake billing_bucket  # Create the Amazon S3 billing Bucket
rake buckets         # Create the Amazon S3 Buckets
rake cost            # Gather costs allocated to the CloudFormation stack f...
rake create          # Create a CloudFormation Stack.
rake delete          # Delete the CloudFormation Stack
rake events          # Get Events from the CloudFormation Stack
rake merge           # Merge the Mapping and UserData sections.
rake merge_mappings  # Merge a Mappings section.
rake merge_userdata  # Merge resource specific Cloud-Init format UserData s...
rake outputs         # Get the Outputs from a CloudFormation Stack
rake parameters      # Create parameters.yml from the template Parameters s...
rake replace         # Delete and then Create a CloudFormation stack.
rake reports         # Download Cost Allocation and Billing Reports
rake stage           # Stage the CloudFormation Template to the S3 Bucket
rake staging_bucket  # Create the Amazon S3 staging Bucket
rake status          # Describe the status of the CloudFormation Stack
rake update          # Update the CloudFormation Stack
rake userdata        # Create Ec2Instance.sh from the Ec2Instance UserData.
rake validate        # Validate the Template with CloudFormation.
```
