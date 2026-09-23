---
name: amazon-lightsail-launch-a-web-server
description: >-
  Launch a reachable Linux web server on Amazon Lightsail — pick a blueprint and bundle, create a key
  pair, create the instance, wait for it to actually be running, open the ports, and give it a static
  IP so the address survives a restart.
api: Amazon Lightsail
apiId: amazon-lightsail:amazon-lightsail-instances-api
generated: '2026-09-17'
method: generated
source: openapi/amazon-lightsail-openapi.yml (derived from smithy/amazon-lightsail-2016-11-28.json)
operations:
  - GetRegions
  - GetBlueprints
  - GetBundles
  - CreateKeyPair
  - CreateInstances
  - GetInstanceState
  - GetInstance
  - OpenInstancePublicPorts
  - GetInstancePortStates
  - AllocateStaticIp
  - AttachStaticIp
  - GetInstanceAccessDetails
  - GetOperation
---

# Launch a web server on Amazon Lightsail

Every call is `POST` to `https://lightsail.{region}.amazonaws.com`, signed with AWS Signature Version 4
for service name `lightsail`, carrying `X-Amz-Target: Lightsail_20161128.<Operation>` and
`Content-Type: application/x-amz-json-1.1`.

## Before you start

- **Pick the Region first and keep it.** Resource names are unique *per Region*, not globally. The same
  instance name in two Regions is two different instances. `GetRegions` lists what is available.
- **There is no idempotency key.** None of the 162 operations accepts one. If `CreateInstances` times
  out and you retry, you can end up paying for two instances. The one guard that works is the name:
  Lightsail rejects a duplicate instance name in the same Region with `InvalidInputException`, so
  choose the name **before** the first attempt and reuse it on every retry.
- **A 200 is not completion.** Most mutating operations return `operation` records. The work is done
  when the operation reaches a terminal status, which you read with `GetOperation`.

## Steps

1. **Choose an image.** `GetBlueprints` returns the OS and application blueprints — `wordpress`,
   `lamp_8_bitnami`, `ubuntu_24_04` and so on. Take the `blueprintId`.
2. **Choose a size.** `GetBundles` returns the priced plans with `bundleId`, `price`, `cpuCount`,
   `ramSizeInGb` and `transferPerMonthInGb`. Take the `bundleId`. `plans/amazon-lightsail-plans-pricing.yml`
   lists the published list prices if you want to reason about cost before calling.
3. **Create an SSH key pair.** `CreateKeyPair` returns the private key **once, in the response body**.
   Store it immediately; it is not retrievable afterwards. Skip this step if you are reusing a key.
4. **Create the instance.** `CreateInstances` with `instanceNames`, `availabilityZone` (a Region plus a
   zone letter, e.g. `us-east-1a`), `blueprintId`, `bundleId` and `keyPairName`. It returns operations,
   not a running server.
5. **Wait.** Poll `GetInstanceState` until `state.name` is `running`, or poll `GetOperation` with the id
   from step 4. Do not proceed on the 200 alone.
6. **Open the ports.** A new instance does not serve HTTP by itself. `OpenInstancePublicPorts` with
   `portInfo` `{fromPort: 80, toPort: 80, protocol: "tcp"}`, and again for 443. Confirm with
   `GetInstancePortStates`.
7. **Give it a stable address.** `AllocateStaticIp` then `AttachStaticIp`. Without this the public IP
   changes when the instance stops and starts — the single most common surprise on this API.
8. **Connect.** `GetInstanceAccessDetails` returns the username, IP and a temporary certificate for
   browser-based SSH. `GetInstance` returns the full resource including `publicIpAddress`.

## Errors you will actually hit

| Exception | HTTP | What it means here |
|---|---|---|
| `InvalidInputException` | 400 | Bad field, or a domain/distribution call sent somewhere other than `us-east-1`, or a duplicate resource name. |
| `NotFoundException` | 404 | Right name, wrong Region — check before assuming the resource is gone. |
| `AccountSetupInProgressException` | 428 | New account still provisioning. Retry. |
| `ThrottlingException` | **400** | Throttled. Note the status: 400, *not* 429. A generic retry handler that treats 4xx as fatal will stop here incorrectly. Back off and retry. |

Full catalogue: `errors/amazon-lightsail-problem-types.yml`.

## Cleaning up

`DeleteInstance` is permanent, and the automatic snapshots attached to the instance are deleted with it.
If you might want it back, take a **manual** snapshot first — see
`amazon-lightsail-back-up-before-you-destroy`.
