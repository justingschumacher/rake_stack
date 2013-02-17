rake_stack
==========

Rake tasks for AWS CloudFormation templates and stacks. 

Clone the project, modify the template, run, and re-run, the Rake tasks.

Features:

* Validate the template with CloudFormation directly
* Merge a common Mappings section into the template
* Extract template Parameters section to a YAML file
* Configuration, Parameters, Outputs, Ids in YAML files
* Create, Update, Delete, a CloudFormation stack

Config:

Add your AWS access keys.

``` bash
vi config.yml
```

Example Usage:

``` bash
rake merge
vi template.json
rake validate
rake parameters
vi parameters.yml
rake create
```
