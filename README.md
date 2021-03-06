# Ansible Cloud Modules & Plugins for AWS Lambda
#### Version 0.7 [![Build Status](https://travis-ci.org/pjodouin/ansible-lambda.svg)](https://travis-ci.org/pjodouin/ansible-lambda)

These modules help manage AWS Lambda resources including code, configuration, aliases, versions, policy statements and
 event source mappings, including DynamoDB and Kinesis streams. A lookup plugin is also included which allows looking
 up values via a Lambda function.


*Note:*  I've cancelled the PR for these modules to be included in 
 _ansible-module-extras_ as it was much too time consuming.  I will continue to maintain them here.

## Requirements
- python >= 2.6
- ansible >= 2.0
- boto3 >= 1.2.3
- importlib (only for running tests on < python 2.7)

## Modules
### lambda_facts:

Gathers facts related to AWS Lambda functions.

##### Example Command
`> ansible localhost -m lambda_facts`

##### Example Playbook
```yaml
# Simple example of listing all info for a function
- name: List all for a specific function
  lambda_facts:
    query: all
    function_name: myFunction
  register: my_function_details
# List all versions of a function
- name: List function versions
  lambda_facts:
    query: versions
    function_name: myFunction
  register: my_function_versions
- name: show Lambda facts
  debug: var=Versions
```
___
### lambda:

Use to create, update or delete lambda functions and publish versions.

##### Example Playbook
```yaml
- hosts: localhost
  gather_facts: no
  vars:
    state: present
    project_folder: /path/to/deployment/package
    deployment_package: lambda.zip
    account: 123456789012
    version_to_delete: 0
  tasks:
  - name: AWS Lambda Function
    lambda:
      state: "{{ state | default('present') }}"
      name: myLambdaFuntion
      publish: True
      description: lambda funtion description
      code_s3_bucket: package-bucket
      code_s3_key: "lambda/{{ deployment_package }}"
      local_path: "{{ project_folder }}/{{ deployment_package }}"
      runtime: python2.7
      timeout: 5
      handler: lambda.handler
      memory_size: 128
      role: "arn:aws:iam::{{ account }}:role/API2LambdaExecRole"
      version: "{{ version_to_delete }}"
      vpc_subnet_ids:
        - subnet-9993085c
        - subnet-99910cc3
      vpc_security_group_ids:
        - sg-999b9ca8
  - name: show results
    debug: var=lambda_facts

```
___
### lambda_alias:

Use to create, update or delete lambda function aliases.

##### Example Playbook
```yaml
- hosts: localhost
  gather_facts: no
  vars:
    state: present
    project_folder: /path/to/deployment/package
    deployment_package: lambda.zip
    account: 123456789012
  tasks:
  - name: AWS Lambda Function
    lambda:
      state: "{{ state | default('present') }}"
      name: myLambdaFuntion
      publish: True
      description: lambda funtion description
      code_s3_bucket: package-bucket
      code_s3_key: "lambda/{{ deployment_package }}"
      local_path: "{{ project_folder }}/{{ deployment_package }}"
      runtime: python2.7
      timeout: 5
      handler: lambda.handler
      memory_size: 128
      role: API2LambdaExecRole
  # The following will set the Dev alias to the latest version ($LATEST) since version is omitted (or = 0)
  - name: "alias 'Dev' for function {{ lambda_facts.FunctionName }} "
    lambda_alias:
      state: "{{ state | default('present') }}"
      function_name: "{{ lambda_facts.FunctionName }}"
      name: Dev
      description: Development is $LATEST version

  # The QA alias will only be created when a new version is published (i.e. not = '$LATEST')
  - name: "alias 'QA' for function {{ lambda_facts.FunctionName }} "
    lambda_alias:
      state: "{{ state | default('present') }}"
      function_name: "{{ lambda_facts.FunctionName }}"
      name: QA
      version: "{{ lambda_facts.Version }}"
      description: "QA is version {{ lambda_facts.Version }}"
    when: lambda_facts.Version != "$LATEST"

```
___
### lambda_event:

Use to create, update or delete lambda function source event mappings, which include Kinesis and DynamoDB streams.
_Note:_  The handling of "push model" events such as S3 events and SNS topics has been removed and should have their
own modules eventually.  Until then, the orginal *lambda_event* module can be found [here.](https://github.com/pjodouin/ansible-repo/tree/master/library)

##### Example Playbook
```yaml
- hosts: localhost
  gather_facts: no
  vars:
    state: present
  tasks:
  - name: DynamoDB stream event mapping
    lambda_event:
      state: "{{ state | default('present') }}"
      event_source: stream
      function_name: "{{ function_name }}"
      alias: Dev
      source_params:
        source_arn: arn:aws:dynamodb:us-east-1:123456789012:table/tableName/stream/2016-03-19T19:51:37.457
        enabled: True
        batch_size: 120
        starting_position: TRIM_HORIZON

  - name: show source event config
    debug: var=lambda_stream_events

```
___
### lambda_policy

This module allows the management of AWS Lambda policy statements. It is idempotent and supports "Check" mode.

##### Example Playbook
```yaml
- hosts: localhost
  gather_facts: no
  vars:
    state: present
  tasks:
  - name: Lambda policy for S3 event notifications
    lambda_policy:
      state: "{{ state | default('present') }}"
      function_name: functionName
      alias: Dev
      statement_id: lambda-s3-myBucket-create-data-log
      action: lambda:InvokeFunction
      principal: s3.amazonaws.com
      source_arn: arn:aws:s3:eu-central-1:123456789012:bucketName
      source_account: 123456789012
#      event_source_token:

```
___
### lambda_invoke:

Use to invoke a specific Lambda function.

##### Example Command
`> ansible localhost -m lambda_invoke -a"function_name=myFunction"`


## Plugins
### lambda (lookup):

Returns value(s) by invoking a lambda function.

##### Example Playbook
```yaml
# Simple example of using a value returned by a lambda function invocation
- hosts: localhost
  gather_facts: no
  vars:
    state: absent
    lambda_output: "{{ lookup('lambda', 'myFunction') }}"
  tasks:
  - name: Test lookup plugin
    debug: var=lambda_output

```

## Installation

Do the following to install the lambda modules in your Ansible environment:

1. Clone this repository or download the ZIP file.

2. Copy the *.py files from the modules directory to your installation custom module directory.  This is, by default, in `./library` which is relative to where your playbooks are located. Refer to the [docs](http://docs.ansible.com/ansible/developing_modules.html#developing-modules) for more information.

3. Copy the *.py files from the plugins directory to your installation custom plugin directory. Custom plugins will go in a directory relating to the plugin type, e.g. a lookup plugin will got into `./lookup_plugins` relative to where your playbooks are located. Refer to the [docs](http://docs.ansible.com/ansible/developing_plugins.html#distributing-plugins) for more information.

4. Make sure boto3 is installed.








