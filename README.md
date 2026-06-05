# AWS Lambda (aws-lambda)

AWS Lambda is a serverless, event-driven compute service that lets you run code for virtually any type of application or backend service without provisioning or managing servers. Lambda runs your code on high-availability compute infrastructure and performs all of the administration of the compute resources, including server and operating system maintenance, capacity provisioning and automatic scaling, and logging.

**APIs.json:** [https://raw.githubusercontent.com/api-evangelist/aws-lambda/refs/heads/main/apis.yml](https://raw.githubusercontent.com/api-evangelist/aws-lambda/refs/heads/main/apis.yml)

## Scope

- **Type:** Index

## Timestamps

- **Created:** 2024-01-15
- **Modified:** 2026-05-19

## APIs

### AWS Lambda API

The AWS Lambda REST API enables you to create, manage, and invoke Lambda functions programmatically. Supports function management, event source mappings, aliases, versions, and layer operations.

- **Human URL:** [https://aws.amazon.com/lambda/](https://aws.amazon.com/lambda/)
- **Base URL:** `https://lambda.{region}.amazonaws.com`

#### Tags

- Compute
- Event-Driven
- FaaS
- Functions
- Serverless

#### Properties

- [Documentation](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [OpenAPI](https://api.apis.guru/v2/specs/amazonaws.com/lambda/2015-03-31/openapi.json) — [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [OpenAPI](openapi/aws-lambda-api-openapi.yml) — [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Postman Collection](collections/aws-lambda-api.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/aws-lambda-api.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)
- [AsyncAPI](asyncapi/aws-lambda-event-triggers-asyncapi.yml) — [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)
- [JSON Schema](json-schema/aws-lambda-function-schema.json) — [JSON Schema](https://json-schema.org/specification)
- [JSON-LD](json-ld/aws-lambda-context.jsonld) — [JSON-LD](https://www.w3.org/TR/json-ld11/)
- [API Reference](https://docs.aws.amazon.com/lambda/latest/api/welcome.html)
- [Pricing](https://aws.amazon.com/lambda/pricing/)
- [Getting Started](https://aws.amazon.com/lambda/getting-started/)
- [SDK](https://aws.amazon.com/tools/)
- [Console](https://console.aws.amazon.com/lambda/)
- [Status Page](https://health.aws.amazon.com/health/status)
- [Rate Limits](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- [Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Security](https://docs.aws.amazon.com/lambda/latest/dg/lambda-security.html)
- [Tutorials](https://aws.amazon.com/lambda/resources/)
- [C L I](https://docs.aws.amazon.com/cli/latest/reference/lambda/)
- [Code Examples](https://docs.aws.amazon.com/lambda/latest/dg/service_code_examples.html)
- [Versioning](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html)
- [Troubleshooting](https://docs.aws.amazon.com/lambda/latest/dg/troubleshooting-deployment.html)

### AWS Lambda Extensions API

The Lambda Extensions API enables you to create extensions that integrate with the Lambda execution environment lifecycle. Extensions can run as companion processes alongside your function, enabling use cases such as capturing diagnostic information, sending telemetry data, and integrating with monitoring and observability tools.

- **Human URL:** [https://docs.aws.amazon.com/lambda/latest/dg/lambda-extensions.html](https://docs.aws.amazon.com/lambda/latest/dg/lambda-extensions.html)
- **Base URL:** `https://lambda.{region}.amazonaws.com`

#### Tags

- Extensions
- Monitoring
- Observability
- Serverless

#### Properties

- [Documentation](https://docs.aws.amazon.com/lambda/latest/dg/lambda-extensions.html)
- [API Reference](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-extensions-api.html)
- [Partners](https://docs.aws.amazon.com/lambda/latest/dg/extensions-api-partners.html)
- [Code Examples](https://github.com/aws-samples/aws-lambda-extensions)
- [Postman Collection](collections/aws-lambda-api.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/aws-lambda-api.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

### AWS Lambda Telemetry API

The Lambda Telemetry API lets you collect telemetry data directly from the Lambda execution environment. Extensions can subscribe to telemetry streams for platform telemetry, function logs, and extension logs to send data to custom destinations for monitoring and observability.

- **Human URL:** [https://docs.aws.amazon.com/lambda/latest/dg/telemetry-api.html](https://docs.aws.amazon.com/lambda/latest/dg/telemetry-api.html)
- **Base URL:** `https://lambda.{region}.amazonaws.com`

#### Tags

- Logging
- Monitoring
- Observability
- Serverless
- Telemetry

#### Properties

- [Documentation](https://docs.aws.amazon.com/lambda/latest/dg/telemetry-api.html)
- [API Reference](https://docs.aws.amazon.com/lambda/latest/dg/telemetry-api-reference.html)
- [Postman Collection](collections/aws-lambda-api.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/aws-lambda-api.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

### AWS Lambda Runtime API

The Lambda Runtime API enables you to use custom runtimes to run functions in any programming language. The runtime API provides an HTTP API for custom runtimes to receive invocation events from Lambda and send response data back within the Lambda execution environment.

- **Human URL:** [https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html)
- **Base URL:** `https://lambda.{region}.amazonaws.com`

#### Tags

- Custom Runtime
- Functions
- Runtime
- Serverless

#### Properties

- [Documentation](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html)
- [API Reference](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html)
- [Postman Collection](collections/aws-lambda-api.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/aws-lambda-api.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

### AWS Lambda Logs API

The Lambda Logs API enables extensions to subscribe to log streams generated by the Lambda platform, function code, and extensions within the execution environment, providing access to log data for processing and forwarding.

- **Human URL:** [https://docs.aws.amazon.com/lambda/latest/dg/runtimes-logs-api.html](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-logs-api.html)
- **Base URL:** `https://lambda.{region}.amazonaws.com`

#### Tags

- Logging
- Monitoring
- Serverless

#### Properties

- [Documentation](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-logs-api.html)
- [Postman Collection](collections/aws-lambda-api.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/aws-lambda-api.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

## Common Properties

- [Terms of Service](https://aws.amazon.com/service-terms/)
- [Privacy Policy](https://aws.amazon.com/privacy/)
- [Blog](https://aws.amazon.com/blogs/compute/category/compute/aws-lambda/)
- [Compliance](https://aws.amazon.com/compliance/)
- [F A Q](https://aws.amazon.com/lambda/faqs/)
- [Partners](https://aws.amazon.com/lambda/partners/)
- [Knowledge Center](https://repost.aws/tags/TA5uNafDy2TpGNjidWLMSxDw/aws-lambda)
- [Changelog](https://docs.aws.amazon.com/lambda/latest/dg/lambda-releases.html)
- [GitHub Repository](https://github.com/awsdocs/aws-lambda-developer-guide)
- [Spectral Rules](rules/aws-lambda-spectral-rules.yml)
- [Vocabulary](vocabulary/aws-lambda-vocabulary.yaml)
- [Features](https://aws.amazon.com/lambda/features/)
- [Use Cases](https://aws.amazon.com/lambda/)
- [Integrations](https://aws.amazon.com/lambda/)

## Maintainers

**FN:** Kin Lane
**Email:** kin@apievangelist.com
