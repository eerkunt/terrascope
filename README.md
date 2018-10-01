# terrascope
A dependency scoping tool for Hashicorp Terraform where you can define dependencies between terraform infrastructure code.

***
Current state of this document is for supporting the development of the tool. User-guide will be written after the tool
is released publicly.
***

# Purpose
The purpose of this tool is to create dependencies between terraform code. Normally one use `terraform modules` in order
 to create re-usable dependencies between the code. In some cases modules can be problematic, like circular dependencies
  between modules. You may find your self creating modules for module consumptions and have to maintain quite a 
  structured multiple repositories. Also, you might have connected services/infrastructures that have dependencies to 
  each other, but completely isolated.

`terrascope` is created to resolve these kind of dependencies where using `terraform modules` may not be applicable due 
to many other reasons. The idea is to basically create YAML files to define those dependencies and run everything once 
in a recursive fashion where things needs to run in order and terraform outputs are shared.

## Quick look
Assume that you have a terraform directory structure like shown in below to define an application. In most cases every 
directory is a separate git repository. 

```
├── env-init
├── providers
│   ├── sandbox
│   │   ├── application_name
│   │   ├── service_name_01
│   │   ├── service_name_02
│   │   └── service_name_03
│   ├── dev
│   │   ├── application_name
│   │   ├── service_name_01
│   │   ├── service_name_02
│   │   └── service_name_03
│   ├── test
│   │   ├── application_name
│   │   ├── service_name_01
│   │   ├── service_name_02
│   │   └── service_name_03
│   └── prod
│   │   ├── application_name
│   │   ├── service_name_01
│   │   ├── service_name_02
│   │   └── service_name_03
├── application_name
│   ├── service_name_01
│   │   ├── firehose
│   │   └── kinesis
│   ├── service_name_02
│   └── service_name_03
```

In a standard way, the way to instantaniate this service (especially in an Enteprise environment where different people 
have different privileges!) is something like ; 

1. Store `terraform.tfvars` and `backend.conf` into `providers/${environment}` to define 
2. Run `env-init` to create IAM roles, KMS keys or anything else required to run with elevated privileges.
3. Run `application_name` to create whatever you need for services. For e.g. ECS Clusters ? 
4. Run the `service_name_**` to create service-related infrastructure.

Usually `application_name` and `service_name_**` directories can be the same directory depending on your application and
 organisational structure. 

In this specific example, assuming an `S3` bucket has been consumed by 2 services at the same time and `kinesis` streams
 (as an terraform output) should be an input for `firehose`. How would you solve this problem ?

Simple.

1. Create `s3` bucket initially on `application_name` layer. 
2. Because you can't use an output of a terraform module as an input to another module (thanks to circular dependency), 
you create either kinesis or firehose as a `resource` not a `module`.
3. You also need to either read the state file via `data` or write shared information that will be used in 
multiple-repos within a key-value store. For e.g. `ssm` ?
4. You usually do these via `Makefile` files living on the root of the repos.

Not even mentioning, you need to clone every repo if they live in different repos.

Instead, with `terrascope` what you do is to create `yml` on the root directories that requires extra dependency. 
Outputs coming from every run is also shared among the other `terraform` runs.

# YAML Structure
`terrascope` requires to have a `scope.yml` or `.scope.yml` file in order to parse and understand which order terraform 
files will be executed.

```yaml
providers:
  name: ${environment} 
  type: <local OR URL or git>
  remote: <path/to/providers OR url/to/providers OR git/repo/to/providers>

order:
  - kinesis
  - firehose
  - name: emre
    type: <local or URL or git>
    remote: <path/to/tffiles OR url/to/tffiles OR git/repo/to/tffiles>
```

In this YAML structure environment variables can be used in `${environment_variable}` format where they will be 
interpolated with the environment variable. If `providers` or `order` elements are in list format, then the value will 
try to be auto-detected.

### providers
Providers as also understandable from the name will define the path where `backend.conf` and `terraform.tfvars` are 
stored that will be passed on any terraform run. The value either can be a path, a URL or a git repository. It can be 
either a map or a list.

Providers will always inherit on sub-directories defined in the `order` list. You don't need to define `providers` again
and again in every sub-directory, assuming you will have a scope in each. If there are some `providers` defined in sub-
directories then it will be overwritten on that specific order and it's childs. 

##### Using different or shared states
As `providers` will inherit on sub-directories defined in the `order` list, it becomes a problem about states in some 
scenarios. On those cases, 

* You can define a separate `providers` block on every sub-directory within the `terrascope.yml`file, 
* You can use the same state defined in parent `backend.conf` that will be inherited into sub-directories,

### order
This section defines the running order of the terraform directories. The values can either be a path, URL or a git 
repository just like in `providers`. All outputs coming from `terraform output` will be feed into 
`providers/terraform.tfvars` temporarily in order to be consumed by other `terraform` directories next in the order list
. The outputs will be have a name like `<order_name>_<output_name> = <output_value>`  

Before running any order element/directory, `terrascope` first check if there is a `scope.yml/.scope.yml` file exist in
the given order directory. If it exists, before running the terraform, first it runs the scope. Thus in a multi-sub
directory structure where there are `scope.yml` files in each, the inner most sub-directory will run first and the root
directory will run last. This will of course create a problem where a sub-directory might have the same name with a 
parent directory in the tree and they both have some `terraform output`s. In this kind of a scenario `terrascope` will
overwrite the existing value dynamically injected into `terraform.tfvars` defined in `providers`, with the running \
order.

Just like in `providers`, environment variables can be used in `${env}` format. 

##### Using shared variables
On the case where you use one `terrascope.yml` on the parent directory, all `terraform output`s will be shared among all
the sub-directories where `terrsacope.yml` lives. Don't forget that the running order will use the inner-most 
sub-directory first.

This may not be applicable sometimes. You may want to share `terraform output`s while using different `terrascope.yml`
and different `state`s defined in your `backend.conf`s. `terrascope` will always share `terraform output`s with a parent
directory, thus you can have different `providers` block on each sub-directory, where `state`s will be isolated but 
variables will be shared among the whole tree in `<order_name>_<output_name> = <output_value>` format.

# Examples
 TBD