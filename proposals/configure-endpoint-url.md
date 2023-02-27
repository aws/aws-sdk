# Service-specific endpoint URL configuration

Document         | Metadata
---------------- | -------------
**Author**       | Kenneth Daily
**Status**       | Draft

## Abstract

This document proposes to extend the options for configuring the endpoint to allow
customers to specify an endpoint URL for individual AWS services. These options
will benefit customers who require more flexibility when configuring the
endpoint used to make requests to AWS services.

## Motivation

Currently, customers can specify the endpoint URL used for AWS requests during
client construction in all AWS SDKs or via the `--endpoint-url` command line
parameter in the AWS CLI. Customers have provided feedback that this does not
allow enough flexibility to support use cases such as:

* Using AWS supported local endpoints like
  [ECS](https://aws.amazon.com/blogs/compute/a-guide-to-locally-testing-containers-with-amazon-ecs-local-endpoints-and-docker-compose/),
  [DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html)
* Using
  [VPC endpoints for STS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_sts_vpce.html)
  ([feature request](https://github.com/aws/aws-cli/issues/6754))
* Using third-party local AWS development environments like
  [LocalStack](https://localstack.cloud/) and
  [ElasticMQ](https://github.com/softwaremill/elasticmq)
* Using private cloud with compatible endpoints [MinIO,](https://min.io/)
  [Ceph Object Gateway](https://docs.ceph.com/en/pacific/radosgw/index.html),
  and Google (AWS DataSync)
* Changing the endpoint used between testing and production

Feedback from customers for these use cases
[spans SDKs and Tools](https://github.com/aws/aws-sdk/issues/229); it has been
noted separately in the JavaScript, PHP, and Python SDKs and the AWS CLI. This
feature is the second most publicly requested feature in the AWS CLI with over
500 upvotes on the GitHub issue, and is in the top ten most requested feature
across all AWS SDKs. An initial draft of this document was published publicly for
customer feedback, and received over 50 upvotes since May 2022. The impact is
most prominent in the AWS CLI where there is no officially supported option for
persisting the endpoint URL; it must always be specified through the command
line parameter `--endpoint-url`.

## Specification

This document defines how SDKs provide service-specific endpoint configuration.
It defines how service-specific endpoint URLs should be read from environment
variables and profiles in the shared configuration file.

### Definitions

- **AWS service identifier:** The unique identifier for a service that is
  exposed in the AWS SDKs and Tools and serves as a reference services across
  SDKs and used for user-facing features.
- **Endpoint URL:** An endpoint URL **MUST** minimally be composed of the scheme
  and host for the endpoint. An endpoint URL **MAY** contain an optional path
  component that can contain one or more path segments.

### Configuration via environment variables

SDKs **MUST** read a customer-provided endpoint URL from an environment
variable. A custom endpoint URL **MUST** be read from a global environment
variable `AWS_ENDPOINT_URL`, which sets the endpoint for requests for all
services. SDKs **MUST** also read service-specific environment variables, which
sets the endpoint for requests to a specific service. The name of the
environment variable **MUST** be in the form `AWS_ENDPOINT_URL_<SERVICE>`, where
`<SERVICE>` is a canonical representation of the service derived from a
standardized transformation of the service ID from the API model for the
service. This transformation **MUST** replace any spaces in the service ID with
underscores and uppercase all letters. There **MUST** be a one-to-one mapping
between AWS service clients provided by the AWS SDKs and environment variable
names. The value of the environment variable will be the endpoint URL.

#### Examples

Setting the global environment variable will result in requests for all services
to use the specified endpoint URL:

```
export AWS_ENDPOINT_URL=https://localhost:4567
```

DynamoDB has a service ID of “DynamoDB”
([link](https://github.com/boto/botocore/blob/bcaf618c4b93c067efa0b85d3e92f3985ff60906/botocore/data/dynamodb/2012-08-10/service-2.json#L10)).
Hence, the endpoint URL environment variable would be set via the following
environment variable:

```
export AWS_ENDPOINT_URL_DYNAMODB=http://localhost:5678
```

As another example, AWS Elastic Beanstalk has a service ID of '“Elastic
Beanstalk”
([link](https://github.com/boto/botocore/blob/bcaf618c4b93c067efa0b85d3e92f3985ff60906/botocore/data/elasticbeanstalk/2010-12-01/service-2.json#L9)).
To set the endpoint for this service, a user would set the following environment
variable:

```
export AWS_ENDPOINT_URL_ELASTIC_BEANSTALK=https://localhost:5567
```

### Configuration via the services definition in the shared configuration file

SDKs **MUST** read service-specific configuration options from a `services`
definition in the
[shared configuration file](https://docs.aws.amazon.com/sdkref/latest/guide/file-format.html).
A `services` definition declares that the attributes that follow (until another
section definition is encountered) are part of a named collection of attributes.
This `services` section is referenced in profile via the `services` parameter to
configure clients for service-specific parameters, including the endpoint URL.

**Format (Configuration Files):**
`[ Whitespace? services Whitespace Identifier Whitespace? ] Whitespace? CommentLine?`

**Format (Credentials Files):** This type of section is not valid in the
credentials file.

**Components:** An services configuration section line
consists of *brackets*, and a *services configuration name*:
`[services-configuration-name]`. A comment line can be appended after the value,
with or without whitespace separating it from the services section definition.

|Component|Description|Type|
|---|---|---|
|Brackets|The `[` and `]` characters at the beginning and end of the services definition|`Operator`|
|---|---|---|
|Services configuration name|An identifier that can be referenced when the SDK is used to refer to this group of properties|`services Identifier`|

In configuration files, the services configuration section name **MUST** start
with `services`. (eg. `[services services-config-name]`). Any amount of
whitespace is permitted between `services` and the name of the configuration
section. The session name **MUST** be explicitly provided. A default `services`
section or default value for a services configuration name **MUST NOT** be
supported. If the services configuration name is empty, the section and all
following properties **MUST** be ignored.

**Examples:**

|Example|Configuration File|Credentials File|
|---|---|---|
|Declaring a services configuration named "my-services-config"|`[services my-services-config]`|Invalid|
|---|---|---|
|Declaring a commented services configuration|`[services my-services-config]; Comment`|Invalid|
|Declaring a services configuration with extra whitespace|`[ \t services \t my-services-config \t ] \t`|Invalid|

The syntax for profile sub-sections has been implemented in the AWS CLI for [S3-specific options](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html).
This convention will be extended to all AWS services. For this document, SDKs
**MUST** read a service-specific endpoint URL from the `endpoint_url` parameter
under the service-specific sub-section. The sub-section name **MUST** be in the
form of `<SERVICE>`, where `<SERVICE>` is a transformation of the AWS service
identifier defined by replacing all spaces with underscores and lowercasing all
letters. There **MUST** be a one-to-one mapping between AWS service clients
provided by the AWS SDKs and sub-profile section names.

#### Examples

This example uses a services definition to configure the global endpoint URL to
use for all services:

```
[services global]
endpoint_url = https://play.min.io:9000

[profile dev-global]
services = global
```

This example uses a `services` definition to configure AWS S3 endpoint URL. The
AWS service identifier is `S3`. Using the specified transformation, the
sub-section name becomes `s3`. The `services` definition is referenced in the
`profile` via the `services` parameter.

```
[services s3-minio]
s3 = 
  endpoint_url = https://play.min.io:9000

[profile dev-minio]
services = s3-minio
```

This example uses a services definition to configure a service-specific endpoint
URL to be used for S3 and a custom global endpoint to be used for all other
services:

```
[services s3-specific-and-global]
endpoint_url = https://localhost:1234
s3 = 
  endpoint_url = https://play.min.io:9000
  
[profile dev-s3-specific-and-global]
services = s3-specific-and-global
```

A single profile can configure an endpoint for more than one
service. This example demonstrates how to set the service-specific endpoint URLs
for AWS S3 and AWS Elastic Beanstalk in the same profile. The AWS service
identifier for AWS Elastic Beanstalk is `Elastic Beanstalk`. Using the specified transformation, the
sub-section name becomes `elastic_beanstalk`.

```
# A profile that sets custom endpoints for more than one service
[services testing-s3-and-eb]
s3 = 
  endpoint_url = https://localhost:4567
elastic_beanstalk = 
  endpoint_url = https://localhost:8000

[profile dev]
services = testing-s3-and-eb
```

The service configuration section is reusable in multiple profiles. For example,
two profiles could use the same `services` definition while altering other
profile properties:

```
[services testing-s3]
s3 = 
  endpoint_url = https://localhost:4567

[profile testing-json]
output = json
services = testing-s3

[profile testing-text]
output = text
services = testing-s3
```

### Resolution order

Setting a service-specific endpoint URL via an environment variable or
configuration option is analogous to setting it via a client or configuration
constructor, and **MUST** be treated by the SDK in the same manner.
Service-specific endpoint configuration **MUST** be resolved with an endpoint
URL provider chain with the following precedence:

1. The value provided through code to an AWS SDK or tool via a command line
   parameter or a client or configuration constructor; for example the
   `--endpoint-url` command line parameter or the `endpoint_url` parameter
   provided to the
   [Python SDK client](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html#boto3.session.Session.client).
2. The value provided by a service-specific environment variable.
3. The value provided by the global endpoint environment variable
   (`AWS_ENDPOINT_URL`).
4. The value provided by a service-specific parameter from a `services`
   definition section in the shared configuration file.
5. The value provided by the global parameter from a `services` definition
   section in the shared configuration file.
6. The value resolved through the methods provided by the SDK or tool when no
   explicit endpoint URL is provided.

If the environment variable or profile configuration value is not specified or
is specified with a blank value, the resolution of the endpoint URL will
continue to the next step defined by the resolution order above. If a non-empty
value is provided, it will be used according to the existing in-code endpoint
configuration logic provided by each individual AWS SDK.

 ### Disabling configuration of endpoint URLs via environment variables and the shared configuration file
 
 SDKs **MUST** read a parameter using the standard parameter resolution logic
 (idiomatic client configuration, environment variable, and shared configuration
 file) boolean option to disable the feature. The client configuration and shared
 configuration profile value **MUST** be named `ignore_config_endpoint_urls`, and
 the environment variable `AWS_IGNORE_CONFIG_ENDPOINT_URLS`.
 
 The default value for this parameter **MUST** be `False`, which results in using
 user-provided endpoints from environment variables or the shared configuration
 file. Changing this option to `True` **MUST** result in not reading any
 configuration option from an environment variable or shared configuration file
 for setting an endpoint URL.

### Publishing specification for service IDs

SDKs **SHOULD** publish the service IDs used for environment variable and
configuration section names. These should be made available to customers on
existing SDK documentation sites. The complete set of service IDs, environment
variable names, and configuration section names **MUST** be published to the
[AWS SDKs and Tools Reference Guide](https://docs.aws.amazon.com/sdkref/latest/guide/overview.html)
and updated whenever a service is launched or changed.

SDKs **MAY** provide an operation for customers to retrieve a list of service
IDs.

## Alternatives

### Alternative 1: Enumerating endpoint URL values for services

We propose to separate the definition of endpoint URLs by providing separate
environment variable names for each service. This approach was chosen because:

1. It’s predicable to determine the required environment variable name for a
   specific AWS service.
2. It’s easy to add, remove, or modify the endpoint URL for a single AWS
   service.
3. The names and values are compatible with all supported operating systems.

Alternatively, we could use a single environment variable with a mapping of
service names to corresponding endpoint URLs. For example:

```
export AWS_ENDPOINT_URL="s3=https://play.min.io:9000,dynamodb=http://localhost:8000“
```

This would be akin to the `PATH` environment variable that concatenates together
multiple paths to search for a command to execute. However, that is only a list,
not a mapping. A drawback to this approach is shell limitations of the length of
values assigned to environment variables. It would also be cumbersome to modify
a single endpoint URL.

### Alternative 2: Specifying configuration options via a file

We propose to use the existing shared configuration file with service-specific
sub-sections to configure the endpoint URL and other service-specific
parameters. Alternatively, we could:

1. Define configuration parameters for each service directly in the profile, as
   is now available in the AWS CLI for configuring S3. For example:

```
[profile development]
aws_access_key_id=foo
aws_secret_access_key=bar
s3 = 
  max_concurrent_requests = 20
  max_queue_size = 10000  
dynamodb = 
  endpoint_url = http://localhost:8000
```

This approach would minimize the changes to syntax used. However, it risks
accidental collisions with existing parameters in the event of a poorly chosen
service name. For example, if a service was released with `region` in the
service ID, this would either directly conflict with a service configuration
name, or risk confusing customers because of the similarity.

1. Define configuration parameters for each service in the profile at the top
   level using a similar naming scheme as the environment variable. For example:

    ```
    # This profile uses the DynamoDB local endpoint
    [profile testing]
    dynamodb_endpoint_url = http://localhost:8000
    ```

This approach would introduce another syntax in addition to the one already
defined. It reduces the readability of the file and makes it more difficult to
make multiple modifications for a specific service. For example, we could
imagine that in the future, DynamoDB could support a dual stack endpoint. With
this alternative, we would expose a parameter similar to the existing s3
parameter for DynamoDB at the top level like:

    ```
    # This profile sets DynamoDB configuration options at the top level
    [profile testing]
    dynamodb_endpoint_url = http://localhost:8000
    dynamodb_use_dualstack_endpoint = true
    ```

We could use other syntaxes for these properties, such as
`dynamodb:endpoint_url` , however these have the same drawbacks. Using a
TOML-like syntax, like `dynamodb.endpoint_url`, would not be compatible with the
AWS CLI because a dot (`.`) has existing meaning with `aws configure` commands.

1. We could use an entirely separate configuration file. All user-configurable
   endpoint options could be stored there. This approach would require all SDKs
   to consume another configuration file and could add confusion for users over
   where configuration options should be stored.

### Alternative 3: Use a different transformation for naming configuration file sections

We propose to transform the AWS service ID by replacing spaces with underscores
and lowercasing all letter to make the profile sub-section name.

Alternatively, we could replace spaces with dashes. This would be more similar
to the existing AWS CLI plugin
[`awscli-plugin-endpoint`](https://github.com/wbingli/awscli-plugin-endpoint),
which uses the AWS CLI command name as profile sub-sections. For example,
setting an endpoint for the Elastic Beanstalk service would look like:

```
[profile local-elasticbeanstalk]
elastic-beanstalk = 
  endpoint_url = https://localhost:8000
```

However, note there still will be differences with the AWS CLI plugin. The
Elastic Beanstalk service has a service ID of “Elastic Beanstalk” and an AWS CLI
command name of `elasticbeanstalk`; the existing AWS CLI plugin would thus use
`elasticbeanstalk` (no dash) as the profile subsection name.

### Alternative 4: Use a new configuration file format

We could use a different configuration file format, like
[TOML](https://toml.io/en/v1.0.0), that has
[broader parsing support](https://github.com/toml-lang/toml/wiki#implementations)
for SDK languages. Using TOML would provide increased syntax flexibility,
portability, and standardization across SDKs. However, it would be a major new
feature for which each SDK would need to add support.

## **Frequently Asked Questions (FAQ)**

### Why use the service ID and not the AWS CLI command name?

There are cases where the service ID differs from the service command name for
the AWS CLI. For example, the service “Elastic Load Balancing v2” has a service
ID of `Elastic Load Balancing v2`, but is invoked in the AWS CLI using the name
`elbv2`. However, all AWS SDKs do not support these names; using the service ID
provides a consistent and automated method to generate the canonical name. The
environment variable name for this service would thus be
`AWS_ENDPOINT_URL_ELASTIC_LOAD_BALANCING_V2`. An exhaustive list of remapped
service names is available
[here](https://github.com/boto/botocore/blob/dd00d7da734c0e5ab7a1cce19573d252efae3dba/tests/functional/test_endpoints.py#L17).

### Why did you chose the specific transformations of the AWS service identifier into environment variable names and profile sub-section names?

We propose to transform the AWS service ID by replacing spaces with underscores
and lowercasing all letter to make the profile sub-section name, and replacing
spaces with underscores and uppercasing all letter to use with the environment
variable name. We chose this for consistency with naming of existing environment
variables and shared configuration variables. It is also easy to describe and
straightforward to implement. See Alternative 4 for more details on why other
transformations were not chosen.

### Will all AWS services be supported with environment variables and configuration sections for the endpoint URL?

Yes, using the defined method to transform the service ID into the supported
environment variable name and profile configuration variable will set the
endpoint to use for that service. All available environment variables and
configuration names will be documented in the
[AWS SDKs and Tools Reference Guide](https://docs.aws.amazon.com/sdkref/latest/guide/overview.html).

### Why should there be a service specific definition in the shared configuration file?

We propose to add a section definition to the shared configuration file to
specifically hold service configuration options like the custom endpoint URL.
Providing a a separate section allows all service configuration options to be
contained together. It also allows reusability across profiles, since a customer
would not need to duplicate all service-specific settings to multiple profiles.

### Why should there be sub-sections in shared configuration services sections to configure service-specific endpoint URLs?

We propose to use service-specific sub-sections to configure the endpoint URL
and other service-specific parameters. This approach was chosen because:

1. There is existing support for the syntax to
   [configure AWS S3](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html)
   with the AWS CLI.
2. All configurations options related to a service are available in one place.
3. This structure is used by the third party implementation of the AWS CLIplugin
   [`awscli-plugin-endpoint`](https://github.com/wbingli/awscli-plugin-endpoint)
   which provides familiarity to others using existing solutions.

### Why should there be support for a global endpoint URL to use for all services?

We propose to configure the endpoint globally in addition to service-specific
custom endpoints. A benefit of using a global endpoint environment variable is
to reduce the risk of accidentally mixing between development and production
endpoints. For example, if a customer is using multiple services each with a
custom endpoint, they may inadvertently not clear them from their environment.
With a single environment variable, this is less of a risk. Some third-party
implementations, like LocalStack, also use a single global endpoint for all
services. In addition, the AWS CLI already provides a mechanism to configure the
endpoint URL for a single operation (through the `--endpoint-url` parameter).

A risk of using a global environment variable to set the endpoint is that it may
break operations that make requests to multiple endpoints. Specifically, this
impacts credential providers like assume role and SSO. The AWS CLI, and possibly
other SDKs, may need to add additional logic to handle this case.

### Why is there a separate parameter to opt out and disable using environment variables or configuration file parameters to set endpoint URLs?

Security-conscious users may wish to disable the feature globally to reduce
risks associated with setting environment variables for the destination of HTTP
traffic. Making the feature opt out via a separate flag allows for this while
also supporting users who want to control the feature outside of code.

The main use case that needs support for this feature is the AWS CLI. If the
parameter is available in code only and users are required to opt in, this would
require all CLI commands to include the parameter. This defeats the purpose of
the feature, since customers can currently specify the endpoint URL on every
call with the `--endpoint-url` parameter. Adding a different required parameter
would defeat the purpose and render the feature unusable for customers.
 