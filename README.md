# Microcks Testcontainers Node

Node/TypeScript library for Testcontainers that enables embedding Microcks into your unit tests with lightweight, throwaway instance thanks to containers

[![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/microcks/microcks-testcontainers-node/build-verify.yml?logo=github&style=for-the-badge)](https://github.com/microcks/microcks-testcontainers-node/actions)
[![Version](https://img.shields.io/npm/v/@microcks/microcks-testcontainers?color=blue&style=for-the-badge)](https://www.npmjs.com/package/@microcks/microcks-testcontainers)
[![License](https://img.shields.io/github/license/microcks/microcks-testcontainers-java?style=for-the-badge&logo=apache)](https://www.apache.org/licenses/LICENSE-2.0)
[![Project Chat](https://img.shields.io/badge/discord-microcks-pink.svg?color=7289da&style=for-the-badge&logo=discord)](https://microcks.io/discord-invite/)

## Build Status

Latest released version is `0.2.2`.

Current development version is `0.2.3`.

## How to use it?

### Include it into your project dependencies

```sh
npm install @microcks/microcks-testcontainers
```

### Startup the container

You just have to specify the container image you'd like to use. This library requires a Microcks `uber` distribution (with no MongoDB dependency).

Version `0.2.2` and above use version `1.8.1` of Microcks container images.

```ts
import { MicrocksContainer } from "@microcks/microcks-testcontainers";

const container = await new MicrocksContainer().start();
```

### Import content in Microcks

To use Microcks mocks or contract-testing features, you first need to import OpenAPI, Postman Collection, GraphQL or gRPC artifacts. 
Artifacts can be imported as main/Primary ones or as secondary ones. See [Multi-artifacts support](https://microcks.io/documentation/using/importers/#multi-artifacts-support) for details.

You can do it before starting the container using arrays of paths:

```ts
import * as path from "path";

const resourcesDir = path.resolve(__dirname, "..", "test-resources");

const container = await new MicrocksContainer()
    .withMainArtifacts([path.resolve(resourcesDir, "apipastries-openapi.yaml")])
    .withSecondaryArtifacts([path.resolve(resourcesDir, "apipastries-postman-collection.json")])
    .start();

```

or once the container is started :

```ts
import * as path from "path";

const resourcesDir = path.resolve(__dirname, "..", "test-resources");

await container.importAsMainArtifact(path.resolve(resourcesDir, "apipastries-openapi.yaml"));
await container.importAsSecondaryArtifact(path.resolve(resourcesDir, "apipastries-postman-collection.json"));
```

Please refer to our [MicrocksContainerTest](https://github.com/microcks/microcks-testcontainers-node/blob/src/microcks-container.test.ts) for comprehensive example on how to use it.

### Using mock endpoints for your dependencies

During your test setup, you'd probably need to retrieve mock endpoints provided by Microcks containers to 
setup your base API url calls. You can do it like this:

```ts
// Get base Url for API Pastries / 0.0.1
var pastriesUrl = container.getRestMockEndpoint("API Pastries", "0.0.1");
```

The container provides methods for different supported API styles/protocols (Soap, GraphQL, gRPC,...).

The container also provides `getHttpEndpoint()` for raw access to those API endpoints.

### Launching new contract-tests

If you want to ensure that your application under test is conformant to an OpenAPI contract (or many contracts),
you can launch a Microcks contract/conformance test using the local server port you're actually running:

```ts
import { MicrocksContainer, TestRequest, TestRunnerType } from "@microcks/microcks-testcontainers";

var testRequest: TestRequest = {
    serviceId: "API Pastries:0.0.1",
    runnerType: TestRunnerType.OPEN_API_SCHEMA,
    testEndpoint: "http://bad-impl:3001",
    timeout: 2000
}

var testResult = await container.testEndpoint(testRequest);

expect(testResult.success).toBe(false);
expect(testResult.testedEndpoint).toBe("http://bad-impl:3001");
expect(testResult.testCaseResults.length).toBe(3);
expect(testResult.testCaseResults[0].testStepResults[0].message).toContain("object has missing required properties");
```

The `TestResult` gives you access to all details regarding success of failure on different test cases.

A comprehensive NestJS demo application illustrating both usages is available here: [nest-order-service](https://github.com/microcks/api-lifecycle/tree/master/shift-left-demo/nest-order-service).

### Using authentication Secrets

It's a common need to authenticate to external systems like Http/Git repositories or external brokers. For that, the `MicrocksContainer` provides the `withSecret()` method to register authentication secrets at startup:

```ts
microcks.withSecret({
        name: 'localstack secret',
        username: 'test',
        password: 'test'
    })
    .start();
```

You may reuse this secret using its name later on during a test like this:

```ts
const testRequest: TestRequest = {
    serviceId: "Pastry orders API:0.1.0",
    runnerType: TestRunnerType.ASYNC_API_SCHEMA,
    testEndpoint: "sqs://us-east-1/pastry-orders?overrideUrl=http://localstack:4566",
    secretName: "localstack secret",
    timeout: 5000
}
```

### Advanced features with MicrocksContainersEnsemble

The `MicrocksContainer` referenced above supports essential features of Microcks provided by the main Microcks container. The list of supported features is the following:

* Mocking of REST APIs using different kinds of artifacts,
* Contract-testing of REST APIs using `OPEN_API_SCHEMA` runner/strategy,
* Mocking and contract-testing of SOAP WebServices,
* Mocking and contract-testing of GraphQL APIs,
* Mocking and contract-testing of gRPC APIs.

To support features like `POSTMAN` contract-testing, we introduced MicrocksContainersEnsemble that allows managing additional Microks services. MicrocksContainersEnsemble allow you to implement [Different levels of API contract testing](https://medium.com/@lbroudoux/different-levels-of-api-contract-testing-with-microcks-ccc0847f8c97) in the Inner Loop with Testcontainers!

A `MicrocksContainersEnsemble` conforms to Testcontaierns lifecycle methods and presents roughly the same interface as a `MicrocksContainer`. To You can create and build an ensemble that way after having initialized a `Network`:

```ts
import { Network } from "testcontainers";
import { MicrocksContainer } from "@microcks/microcks-testcontainers";

const network = await new Network().start();

const ensemble = await new MicrocksContainersEnsemble(network)
    .withMainArtifacts([path.resolve(resourcesDir, "apipastries-openapi.yaml")])
    .withSecondaryArtifacts([path.resolve(resourcesDir, "apipastries-postman-collection.json")])
    .start();
```

A `MicrocksContainer` is wrapped by an ensemble and is still available to import artifacts and execute test methods. You have to access it using:

```ts
const microcks = ensemble.getMicrocksContainer();
await microcks.importAsMainArtifact(...);
(await microcks.logs())
  .on("data", line => console.log(line))
  .on("err", line => console.error(line))
  .on("end", () => console.log("Stream closed"));
```

Please refer to our [MicrocksContainersEnsembleTest](https://github.com/microcks/microcks-testcontainers-node/blob/src/microcks-containers-ensemble.test.ts) for comprehensive example on how to use it.

#### Postman contract-testing

On this `ensemble` you may want to enable additional features such as Postman contract-testing:

```ts
ensemble.withPostman();
ensemble.start();
```

You can execute a `POSTMAN`` test using an ensemble that way:

```ts
var testRequest = {
    serviceId: "API Pastries:0.0.1",
    runnerType: TestRunnerType.POSTMAN,
    testEndpoint: "http://good-impl:3003",
    timeout: 3000
}

var testResult = await ensemble.getMicrocksContainer().testEndpoint(testRequest);

expect(testResult.success).toBe(true);
expect(testResult.testedEndpoint).toBe("http://good-impl:3003");
expect(testResult.testCaseResults.length).toBe(3);
expect(testResult.testCaseResults[0].testStepResults[0].message).toBeUndefined();
```

#### Asynchronous API support

Asynchronous API feature need to be explicitly enabled as well. In the case you want to use it for mocking purposes,
you'll have to specify additional connection details to the broker of your choice. See an example below with connection
to a Kafka broker:

```ts
ensemble.withAsyncFeature()
    .withKafkaConnection({
        bootstrapServers: "kafka:9092"
    });
ensemble.start();
```

##### Using mock endpoints for your dependencies

Once started, the `ensemble.getAsyncMinionContainer()` provides methods for retrieving mock endpoint names for the different
supported protocols (WebSocket, Kafka, SQS and SNS).

```ts
const kafkaTopic = ensemble.getAsyncMinionContainer()
    .getKafkaMockTopic("Pastry orders API", "0.1.0", "SUBSCRIBE pastry/orders");
```

##### Launching new contract-tests

Using contract-testing techniques on Asynchronous endpoints may require a different style of interacting with the Microcks
container. For example, you may need to:
1. Start the test making Microcks listen to the target async endpoint,
2. Activate your System Under Tests so that it produces an event,
3. Finalize the Microcks tests and actually ensure you received one or many well-formed events.

As the `MicrocksContainer` provides the `testEndpoint(request: TestRequest): Promise<TestResult>` method, this is actually easy like:

```ts
// Start the test, making Microcks listen the endpoint provided in testRequest
let testResultPromise: Promise<TestResult> = ensemble.getMicrocksContainer().testEndpoint(testRequest);

// Here below: activate your app to make it produce events on this endpoint.
// myapp.invokeBusinessMethodThatTriggerEvents();

// Now retrieve the final test result and assert.
let testResult = await testResultPromise;
expect(testResult.success).toBe(true);
```
