# terrascope
A dependency scoping tool for Hashicorp Terraform where you can define dependencies between terraform infrastructure code.

# Purpose
The purpose of this tool is to create dependencies between terraform code. Normally one use `terraform modules` in order to create re-usable dependencies between the code. In some cases modules can be problematic, like circular dependencies between modules. You may find your self creating modules for module consumptions and have to maintain quite a structured multiple repositories. Also, you might have connected services/infrastructures that have dependencies to each other, but completely isolated.

`terrascope` is created to resolve these kind of dependencies where using `terraform modules` may not be applicable due to many other reasons. The idea is to basically create YAML files to define those dependencies and run everything once in a recursive fashion where things needs to run in order and terraform outputs are shared.

## Example
Assume that you have a terraform directory structure like shown in below to define an application. In most cases every directory is a separate git repository. 

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

In a standard way, the way to instantaniate this service (especially in an Enteprise environment where different people have different privileges!) is something like ; 

1. Store `terraform.tfvars` and `backend.conf` into `providers/${environment}` to define 
2. Run `env-init` to create IAM roles, KMS keys or anything else required to run with elevated privileges.
3. Run `application_name` to create whatever you need for services. For e.g. ECS Clusters ? 
4. Run the `service_name_**` to create service-related infrastructure.

Usually `application_name` and `service_name_**` directories can be the same directory depending on your application and organisational structure. 

In this specific example, assuming an `S3` bucket has been consumed by 2 services at the same time and `kinesis` streams (as an terraform output) should be an input for `firehose`. How would you solve this problem ?

Simple.

1. Create `s3` bucket initially on `application_name` layer. 
2. Because you can't use an output of a terraform module as an input to another module (thanks to circular dependency), you create either kinesis or firehose as a `resource` not a `module`.
3. You also need to either read the state file via `data` or write shared information that will be used in multiple-repos within a key-value store. For e.g. `ssm` ?
4. You usually do these via `Makefile` files living on the root of the repos.

Not even mentioning, you need to clone every repo if they live in different repos.

Instead, with `terrascope` what you do is to create `yml` on the root directories that requires extra dependency. Outputs coming from every run is also shared among the other `terraform` runs.

# YAML Structure

TBD.
