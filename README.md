[![Build Status](https://defradev.visualstudio.com/DEFRA_FutureFarming/_apis/build/status/defra-ffc-demo-calculation-service?branchName=master)](https://defradev.visualstudio.com/DEFRA_FutureFarming/_build/latest?definitionId=612&branchName=master)
[![Known Vulnerabilities](https://snyk.io//test/github/DEFRA/ffc-demo-calculation-service/badge.svg?targetFile=package.json)](https://snyk.io//test/github/DEFRA/ffc-demo-calculation-service?targetFile=package.json)

# FFC Demo Calculation Service

Digital service mock to claim public money in the event property subsides into mine shaft.  The calculation service subscribes to a message queue for new claims and calculates a value for each claim.  Once calculated it publishes the value to a message queue.

## Prerequisites

Either:
- Docker
- Docker Compose

Or:
- Kubernetes
- Helm

Or:
- Node 10
- AMQP 1.0 message queue

## Environment variables

The following environment variables are required by the application container. Values for development are set in the Docker Compose configuration. Default values for production-like deployments are set in the Helm chart and may be overridden by build and release pipelines.

 | Name                       | Description                 | Required | Default     | Valid                               | Notes |
 |----------------------------|-----------------------------|:--------:|-------------|-------------------------------------|-------|
 | NODE_ENV                   | Node environment            | no       | development | development,test,production         |       |
 | MESSAGE_QUEUE_HOST         | Message queue host          | no       |             | myservicebus.servicebus.windows.net |       |
 | MESSAGE_QUEUE_PORT         | Message queue port          | no       |             | 5671,5672                           |       |
 | MESSAGE_QUEUE_TRANSPORT    | Message queue transport     | yes      | tcp         | tcp,ssl                             | standard port is 5671 for ssl, 5672 for tcp |
 | CALCULATION_QUEUE_ADDRESS  | calculation queue name      | no       |             | calculation                         |       |
 | CALCULATION_QUEUE_USER     | calculation queue user name | no       |             |                                     |       |
 | CALCULATION_QUEUE_PASSWORD | calculation queue password  | no       |             |                                     |       |
 | PAYMENT_QUEUE_ADDRESS      | payment queue name          | no       |             | payment                             |       |
 | PAYMENT_QUEUE_USER         | payment queue user name     | no       |             |                                     |       |
 | PAYMENT_QUEUE_PASSWORD     | payment queue password      | no       |             |                                     |       |
 | HEALTHZ_FILE_INTERVAL_IN_MILLIS | Interval for creation of healthz file | no | 10000 |                                  | Maximum value 30000  |

## How to run tests

A convenience script is provided to run automated tests in a containerised environment. The first time this is run, container images required for testing will be automatically built. An optional `--build` (or `-b`) flag may be used to rebuild these images in future (for example, to apply dependency updates).

```
# Run tests
scripts/test

# Rebuild images and run tests
scripts/test --build
```

This runs tests via a `docker-compose run` command. If tests complete successfully, all containers, networks and volumes are cleaned up before the script exits. If there is an error or any tests fail, the associated Docker resources will be left available for inspection.

Alternatively, the same tests may be run locally via npm:

```
# Run tests without Docker
npm run test
```

Running the integration tests locally requires a message bus that supports AMQP 1.0 and the following environment variables setting:
`MESSAGE_QUEUE_HOST`, `MESSAGE_QUEUE_PORT`, `CALCULATION_QUEUE_USER`, `CALCULATION_QUEUE_PASSWORD`, `PAYMENT_QUEUE_ADDRESS`, `PAYMENT_QUEUE_USER`, `PAYMENT_QUEUE_PASSWORD`

## Running the application

The application is designed to run in containerised environments, using Docker Compose in development and Kubernetes in production.

- Scripts are provided to aid local development and testing using Docker Compose.
- A Helm chart is provided for production deployments to Kubernetes.

### Build container image

Container images are built using Docker Compose, with the same images used to run the service with either Docker Compose or Kubernetes.

By default, the start script will build (or rebuild) images so there will rarely be a need to build images manually. However, this can be achieved through the Docker Compose [build](https://docs.docker.com/compose/reference/build/) command:

```
# Build container images
docker-compose build
```

### Start and stop the service

Use the provided [`start`](./scripts/start) and [`stop`](./scripts/stop) scripts to run the service locally via Docker Compose. Both scripts accept a number of flags to customise their behaviour. For full instructions on the flags available to each script, use the `--help` or `-h` flag:

```
# View instructions for the start script
scripts/start --help

# View instructions for the stop script
scripts/stop --help
```

By default, the start script will build new container images before starting the service on an isolated Docker network along with any direct dependencies, such as message queues and databases. It will not automatically replace existing containers or volumes, but will warn if there is a conflict and abort the request. Use the `--clean` or `--quick` flags to instruct the script to replace or keep existing resources, respectively.

The underlying `docker-compose up/down` commands can be customised by appending `-- [DOCKER_COMPOSE_ARGS]` after any other arguments to the `start/stop` scripts. For example:

```
# Start the service without attaching to logs
scripts/start -- --detach
```

### Test the service

This service reacts to messages retrieved from an AMQP 1.0 message broker so manual testing involves pushing messages into the appropriate message queue. The [start](./scripts/start) script runs [ActiveMQ Artemis](https://activemq.apache.org/components/artemis) alongside the application to provide the required message bus and broker.

The [send-test-mesage](./scripts/send-test-message) script will push a valid message into one of the Artemis queues which the app subscribes to. The container logs should show the message being picked up and processed automatically when the following command is executed.

```
# Send a sample message to the Artemis message queue
scripts/send-test-message
```

For more detailed testing, messages can be pushed into queues via the Artemis console UI hosted at http://localhost:8161/console/login (username: artemis, password: artemis). Messages should match the format of the sample JSON below.

```
{
  "claimId": "MINE123",
  "propertyType": "business",
  "accessible": false,
  "dateOfSubsidence": "2019-07-26T09:54:19.622Z",
  "mineType": ["gold"],
  "email": "test@email.com"
}
```

### Link to sibling services

To test interactions with sibling services in the FFC demo application, it is necessary to connect each service to an external Docker network, along with shared dependencies such as message queues. The most convenient approach for this is to start the entire application stack from the [`ffc-demo-development`](https://github.com/DEFRA/ffc-demo-development) repository.

It is also possible to run a limited subset of the application stack, using the [`start`](./scripts/start) script's `--link` flag to join each service to the shared Docker network. See the [`ffc-demo-development`](https://github.com/DEFRA/ffc-demo-development) Readme for instructions.

### Deploy to Kubernetes

For production deployments, a helm chart is included in the `.\helm` folder. This service connects to an AMQP 1.0 message broker, using credentials defined in [values.yaml](./helm/ffc-demo-calculation-service/values.yaml), which must be made available prior to deployment.

Scripts are provided to test the Helm chart by deploying the service, along with an appropriate message broker, into the current Helm/Kubernetes context.

```
# Deploy to current Kubernetes context
scripts/helm/install

# Remove from current Kubernetes context
scripts/helm/delete
```

#### Probes

The service has a command based liveness probe.  The probe will write a file containing a timestamp to the `/tmp` directory.  If the timestamp has not been updated in the last 30 seconds, the probe will report the service is not functioning.

## Dependency management

Dependencies should be managed within a container using the development image for the app. This will ensure that any packages with environment-specific variants are installed with the correct variant for the contained environment, rather than the host system which may differ between development and production.

The [`exec`](./scripts/exec) script is provided to run arbitrary commands, such as npm, in a running service container. If the service is not running when this script is called, it will be started for the duration of the command and then removed.

Since dependencies are installed into the container image, a full build should always be run immediately after any dependency change.

In development, the `node_modules` folder is mounted to a named volume. This volume must be removed in order for dependency changes to propagate from the rebuilt image into future instances of the app container. The [`start`](./scripts/start) script has a `--clean` (or `-c`) option  which will achieve this.

The following example will update all npm dependencies, rebuild the container image and replace running containers and volumes:

```
# Run the NPM update
scripts/exec npm update

# Rebuild and restart the service
scripts/start --clean
```

## Build pipeline

The [azure-pipelines.yaml](azure-pipelines.yaml) performs the following tasks:
- Runs unit tests
- Publishes test result
- Pushes containers to the registry tagged with the PR number or release version
- Deletes PR deployments, containers, and namepace upon merge

Builds will be deployed into a namespace with the format `ffc-demo-calculation-service-{identifier}` where `{identifier}` is either the release version, the PR number, or the branch name.

A detailed description on the build pipeline and PR work flow is available in the [Defra Confluence page](https://eaflood.atlassian.net/wiki/spaces/FFCPD/pages/1281359920/Build+Pipeline+and+PR+Workflow)

## License

THIS INFORMATION IS LICENSED UNDER THE CONDITIONS OF THE OPEN GOVERNMENT LICENCE found at:

<http://www.nationalarchives.gov.uk/doc/open-government-licence/version/3>

The following attribution statement MUST be cited in your products and applications when using this information.

> Contains public sector information licensed under the Open Government license v3

### About the license

The Open Government Licence (OGL) was developed by the Controller of Her Majesty's Stationery Office (HMSO) to enable information providers in the public sector to license the use and re-use of their information under a common open licence.

It is designed to encourage use and re-use of information freely and flexibly, with only a few conditions.
