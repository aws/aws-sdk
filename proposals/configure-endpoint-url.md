# [proposal] Configuration of service-specific endpoint URLs

## Abstract

Users have provided feedback that more flexibility is needed to configure the endpoint used when making requests to AWS services. This document proposes to extend the options for configuring the endpoint to allow users to provide an endpoint URL independently for each AWS service via an environment variable or a profile subsection in the shared configuration file.

## Motivation

Currently, users can specify the endpoint URL used for AWS requests during client construction in all AWS SDKs or via the `--endpoint-url` command line parameter in the AWS CLI. Users have provided feedback that this does not allow enough flexibility to support use cases such as:

* Using AWS supported local endpoints like [ECS](https://aws.amazon.com/blogs/compute/a-guide-to-locally-testing-containers-with-amazon-ecs-local-endpoints-and-docker-compose/), [DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html), and [VPC endpoints for STS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_sts_vpce.html)
* Using third-party local AWS development environments like [LocalStack](https://localstack.cloud/) and [ElasticMQ](https://github.com/softwaremill/elasticmq)
* Using private cloud with compatible endpoints like [MinIO](https://min.io/) and [Ceph Object Gateway](https://docs.ceph.com/en/pacific/radosgw/index.html)
* Changing the endpoint used between testing production

Feedback for these use case spans SDKs and Tools; it has been noted separately in the JavaScript, PHP, and Python SDKs and the AWS CLI. The impact is most prominent in the AWS CLI where there is no officially supported option for persisting the endpoint URL; it must always be specified through the command line parameter `--endpoint-url`.

For example, one user commented:


>This should be implemented on the same level as you can provide credentials and region. Either as environment variables or in a profile. I am struggling setting up a docker based environment where local offline development goes towards localstack and production goes towards AWS. Containers running the same image, just configured via environment variables.

>*Originally posted by @johansmolinski in https://github.com/aws/aws-cli/issues/4454#issuecomment-709509072*


There are existing customer implementations to provide this feature via the [AWS CLI plugin](https://github.com/wbingli/awscli-plugin-endpoint), [third party packages](https://github.com/localstack/awscli-local), *ad hoc* environment variables, and shell scripts shared in GitHub issues.

Adding this functionality would alleviate significant customer friction for common use cases where the endpoint needs to be changed.

## Proposed Interface

We propose that users can configure the endpoint URL used for each service separately. In addition to the existing configuration options, we will add the ability to configure the endpoint URL via environment variables and the shared configuration file.

### Configuration via environment variables

Service-specific environment variable names  to set the endpoint URL will be `AWS_<SERVICE>_ENDPOINT_URL`, where `<SERVICE>` is a cannonical representation of the service derived from a standardized transformation of the service ID from the API model for the service. This transformation will replace any spaces in the service ID with underscores and uppercase all letters. There is a one-to-one mapping between AWS service clients provided by the AWS SDKs and environment variable names.

#### Examples

For example, DynamoDB has a service ID of “DynamoDB” ([link](https://github.com/boto/botocore/blob/bcaf618c4b93c067efa0b85d3e92f3985ff60906/botocore/data/dynamodb/2012-08-10/service-2.json#L10)). Hence, the endpoint URL environment variable would be set via the following environment variable:

```
export AWS_DYNAMODB_ENDPOINT_URL=http://localhost:5678
```

As another example, AWS Elastic Beanstalk has a service ID of '“Elastic Beanstalk” ([link](https://github.com/boto/botocore/blob/bcaf618c4b93c067efa0b85d3e92f3985ff60906/botocore/data/elasticbeanstalk/2010-12-01/service-2.json#L9)). To set the endpoint for this service, a user would set the following environment variable:

```
export AWS_ELASTIC_BEANSTALK_ENDPOINT_URL=https://localhost:5567
```

### Shared configuration file

A profile in the [shared configuration file](https://docs.aws.amazon.com/sdkref/latest/guide/file-format.html) will accept sub-sections to configure specific AWS services. The syntax for profile sub-sections has been previously defined: the AWS CLI supports a profile sub-section for AWS S3 ([documentation](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html)). This convention will be extended to other services and hold a parameter for the configurable endpoint URL. Similar to the the environment variables, the sub-section name will be defined by replacing all spaces with underscores and lowercasing all letters in the service ID. There is a one-to-one mapping between AWS service clients provided by the AWS SDKs and sub-profile section names. 

#### Examples

The following shared configuration profiles make use of the existing `s3` sub-section and new Elastic Beanstalk sub-sections to configure their respective endpoint URLs.

```
# This profile uses the existing s3 sub-section to change the s3 endpoint
[profile s3-minio]
s3 = 
  endpoint_url = https://play.min.io:9000

# This profile uses a custom Elastic Beanstalk local endpoint
[profile local-elasticbeanstalk]
elastic_beanstalk = 
  endpoint_url = https://localhost:8000
```

Similarly, a single profile could configure the endpoints for more than one service, providing a way to switch between a testing and production environment with a separate profile. In the following example, the testing profile would use `localhost` for S3 and DynamoDB operations, while the production profile would use normal AWS endpoints:

```
# A profile that sets custom endpoints for more than one service
[profile testing]
output = json
s3 = 
  endpoint_url = https://localhost:4567
dynamodb = 
  endpoint_url = https://localhost:5678

# A profile that uses default endpoints or those defined through
# other endpoint configuration providers
[profile production]
output = json
```

### Resolution order

These new configuration options will be resolved with an endpoint URL provider chain with the following precedence:

1. The value provided through code to an AWS SDK or tool via a command line parameter or a client or configuration constructor; for example the  `--endpoint-url` command line parameter or the `endpoint_url` parameter provided to the [Python SDK client](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html#boto3.session.Session.client).
2. The value provided by a service-specific environment variable.
3. The value provided by a service-specific parameter from the shared configuration file.
4. The value resolved through the methods provided by the SDK or tool when no explicit endpoint URL is provided.

If the environment variable or profile configuration value is specified with an empty string or a blank value, the resolution of the endpoint URL will continue to the next step defined by the resolution order above. If a non-empty value is provided, it will be used according to the existing in-code endpoint configuration logic provided by each individual AWS SDK.

## Alternatives

### Enumerating endpoint URL values for services

We propose to separate the definition of endpoint URLs by providing separate environment variable names for each service. This approach was chosen because:

1. It’s predicable to determine the required environment variable name for a specific AWS service.
2. It’s easy to add, remove, or modify the endpoint URL for a single AWS service.
3. The names and values are compatible with all supported operating systems.

Alternatively, we could use a single environment variable with a mapping of service names to corresponding endpoint URLs. For example:

```
export AWS_ENDPOINT_URL="s3=https://play.min.io:9000,dynamodb=http://localhost:8000“
```

This would be akin to the `PATH` environment variable that concatenates together multiple paths to search for a command to execute. However, that is only a list, not a mapping. A drawback to this approach is shell limitations of the length of values assigned to environment variables. It would also be cumbersome to modify a single endpoint URL.

### Set a global endpoint URL for all services

We propose to configure the endpoint per service. This satisfies the most common use case where most services will use standard AWS endpoints but specific services use an alternative endpoint.

Alternatively, we could set the endpoint URL globally for all services. This could be exposed through a single environment variable, like:

```
AWS_ENDPOINT_URL=http://localhost:5678
```

Or, a top-level configuration option, like:

```
# This profile would use the specified endpoint for all requests
[profile global-endpoint]
endpoint_url = http://localhost:1234
```

This would have a high likelihood of confusion and broken calls for users who make requests to multiple services that have different endpoints, or require users to switch profiles to manage the logic for using different endpoints. Should it be required, this could be added after service specific endpoints.

### Specifying configuration options via a file

We propose to use the existing shared configuration file with sub-sections to configure the endpoint URL and other endpoint-related parameters. This approach was chosen because:

1. There is existing support for the syntax to [configure AWS S3](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html) with the AWS CLI.
2. All configurations options related to a service are available in one place.
3. This structure is used by the third party implementation of the AWS CLI plugin [`awscli-plugin-endpoint`](https://github.com/wbingli/awscli-plugin-endpoint) which provides familiarity to others using existing solutions.

Alternatively, we could:

1. Define configuration parameters for each service in the profile at the top level using a similar naming scheme as the environment variable. For example:

    ```
    # This profile uses the AWS standard endpoint for the defined region
    [profile production]
    region=us-west-2

    # This profile uses the DynamoDB local endpoint
    [profile testing]
    dynamodb_endpoint_url = http://localhost:8000
    ```

    This approach would result in a cluttered profile, especially if more configuration options are added for a service. This would reduce the readability of the file and make it more difficult to make multiple modifications for a specific service. For example, we could imagine that in the future, DynamoDB could support a dual stack endpoint. With this alternative, we would expose a parameter similar to the existing s3 parameter for DynamoDB at the top level like:

    ```
    # This profile sets DynamoDB configuration options at the top level
    [profile testing]
    dynamodb_endpoint_url = http://localhost:8000
    dynamodb_use_dualstack_endpoint = true
    ```

2. We could use an entirely separate configuration file. All user-configurable endpoint options could be stored there. This approach would require all SDKs to consume another configuration file and could add confusion for users over where configuration options should be stored.

### Use a different transformation for naming configuration file sections

We propose to transform the AWS service ID by replacing spaces with underscores and lowercasing all letter to make the profile sub-section name. We chose this for consistency with naming of the environment variables and other existing shared configuration variables.

Alternatively, we could replace spaces with dashes. This would be more similar to the existing AWS CLI plugin [`awscli-plugin-endpoint`](https://github.com/wbingli/awscli-plugin-endpoint), which uses the AWS CLI command name as profile sub-sections. For example, setting an endpoint for the Elastic Beanstalk service would look like:

```
[profile local-elasticbeanstalk]
elastic-beanstalk = 
  endpoint_url = https://localhost:8000
```

However, note there still will be differences with the AWS CLI plugin. The Elastic Beanstalk service has a service ID of “Elastic Beanstalk” and an AWS CLI command name of `elasticbeanstalk`; the existing AWS CLI plugin would thus use `elasticbeanstalk` (no dash) as the profile subsection name.



## **Frequently Asked Questions (FAQ)**

### Q. Why use the service ID and not the AWS CLI command name?

There are cases where the service ID differs from the service command name for the AWS CLI. For example, the service “Elastic Load Balancing v2” has a service ID of `Elastic Load Balancing v2`, but is invoked in the AWS CLI using the name `elbv2`. However, all AWS SDKs do not support these names; using the service ID provides a consistent and automated method to generate the canonical name. The environment variable name for this service would thus be `AWS_ELASTIC_LOAD_BALANCING_V2_ENDPOINT_URL`. An exhaustive list of remapped service names is available [here](https://github.com/boto/botocore/blob/dd00d7da734c0e5ab7a1cce19573d252efae3dba/tests/functional/test_endpoints.py#L17).

### Q. Will all AWS services be supported with environment variables and configuration sections for the endpoint URL?

Yes, using the defined method to transform the service ID into the supported environment variable name and profile configuration variable will set the endpoint to use for that service. All available environment variables and configuration names will be documented in the [AWS SDKs and Tools Reference Guide](https://docs.aws.amazon.com/sdkref/latest/guide/overview.html).
