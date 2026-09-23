# Superseded OpenAPI

`amazon-lightsail-instances-api-openapi.yml` was a six-operation, hand-authored scaffold that modelled
Lightsail as a REST API with paths like `/instances` and `/instances/{instanceName}/start`. Lightsail
does not work that way: the wire protocol is AWS JSON 1.1 and the real paths, which AWS declares in its
own Smithy service model, are `/ls/api/2016-11-28/<Operation>`.

It was superseded on 2026-09-17 by `openapi/amazon-lightsail-openapi.yml`, a mechanical transform of
the AWS-published Smithy 2.0 service model (`smithy/amazon-lightsail-2016-11-28.json`, from
github.com/aws/api-models-aws) carrying all 162 operations with their real paths, request and response
schemas, and declared errors.

Kept here for provenance only. It is not a source document: nothing should read, refine, split or
register it.
