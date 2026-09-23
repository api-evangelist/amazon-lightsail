---
name: amazon-lightsail-deploy-a-container-service
description: >-
  Deploy a container to Amazon Lightsail — create the service, get an image into it, ship a deployment,
  and read the logs when it does not come up. The only Lightsail surface that is REST-shaped rather than
  AWS-JSON-shaped, and the only one that needs a separate local binary.
api: Amazon Lightsail
apiId: amazon-lightsail:amazon-lightsail-instances-api
generated: '2026-09-17'
method: generated
source: openapi/amazon-lightsail-openapi.yml (derived from smithy/amazon-lightsail-2016-11-28.json)
operations:
  - CreateContainerService
  - GetContainerServices
  - GetContainerServicePowers
  - CreateContainerServiceRegistryLogin
  - RegisterContainerImage
  - GetContainerImages
  - CreateContainerServiceDeployment
  - GetContainerServiceDeployments
  - GetContainerLog
  - GetContainerServiceMetricData
  - UpdateContainerService
  - DeleteContainerService
---

# Deploy a container service on Amazon Lightsail

Container services are the exception to every shape rule on this API. While the other 151 operations are
`POST` with an `X-Amz-Target` header, this subsurface has real REST paths and real verbs:

```
GET    /ls/api/2016-11-28/container-services/{serviceName}/images
POST   /ls/api/2016-11-28/container-services/{serviceName}/deployments
PATCH  /ls/api/2016-11-28/container-services/{serviceName}
DELETE /ls/api/2016-11-28/container-services/{serviceName}
GET    /ls/api/2016-11-28/container-services/{serviceName}/containers/{containerName}/log
```

It is still SigV4-signed, and the AWS-JSON operations still exist for the same resources — the model
carries both shapes.

## Steps

1. **Create the service.** `CreateContainerService` with `serviceName`, `power` (nano → xlarge, list
   them with `GetContainerServicePowers`) and `scale` (node count). Poll `GetContainerServices` until
   `state` is `READY`. Creation takes minutes, not seconds.
2. **Get an image in.** Two routes:
   - *Public registry* — skip registration entirely and name the image directly in the deployment
     (`nginx:latest`, an ECR Public URI).
   - *Local image* — this is where a second binary is required. `aws lightsail push-container-image` is
     a CLI-only command implemented by the **lightsailctl** plugin; it uploads the local image and
     registers it. There is no pure-API equivalent for uploading bytes. `RegisterContainerImage` expects
     an image that is already in place, and `CreateContainerServiceRegistryLogin` returns short-lived
     Docker credentials. Confirm with `GetContainerImages`.
3. **Deploy.** `CreateContainerServiceDeployment` with a `containers` map (image, command, environment,
   ports) and a `publicEndpoint` naming which container and port is public, plus its health check. Each
   call creates a new numbered deployment version; `GetContainerServiceDeployments` lists them, and the
   service quota is 50 versions.
4. **Watch it come up.** Poll `GetContainerServiceDeployments` until the newest version is `ACTIVE`. A
   failing health check leaves it `FAILED` and the previous version keeps serving.
5. **Debug.** `GetContainerLog` with `serviceName`, `containerName` and an optional `filterPattern` and
   time range. `GetContainerServiceMetricData` gives CPU and memory.

## Things that will bite

- **Scaling is an update, not a redeploy.** `UpdateContainerService` changes `power` and `scale` in
  place; it does not need a new deployment.
- **Quotas are per Region:** 100 container services, 20 nodes per service, 10 containers per deployment,
  50 deployment versions, 150 stored images, 4 custom domains, 4 certificates, and container logs are
  kept 4 days. Full list in `rate-limits/amazon-lightsail-rate-limits.yml`.
- **No rollback operation exists.** To go back, create a new deployment with the previous version's
  container definition — read it from `GetContainerServiceDeployments` first, because
  `DeleteContainerService` takes the whole history with it.
- **No idempotency key.** A retried `CreateContainerServiceDeployment` creates a second deployment
  version and burns one of the 50.
