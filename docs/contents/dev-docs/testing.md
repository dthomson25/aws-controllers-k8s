# Testing

For local development and testing we use [kind](https://kind.sigs.k8s.io/), 
which in turn requires Docker. To build and test an ACK controller with a
`kind` cluster, execute the commands as described in the following from the
root directory of your [checked-out source repository](../setup/).

!!! warning "Footprint"
    When you run the `scripts/kind-build-test.sh` script the first time,
    the step that builds the container image for the target ACK service
    controller can 40 or more minutes. This is because the container image
    contains a lot of dependencies. Once you successfully build the target
    image this base image layer is cached locally and the build takes a much 
    shorter amount of time. We are aware of this (and the storage footprint,
    ca. 3 GB) and aim to reduce both in the fullness of time.

## Preparation

To build the latest `ack-generate` binary, execute the following command:

```
make build-ack-generate
```

Don't worry if you forget this, the script in the next step will complain with
an `ERROR: Unable to find an ack-generate binary` message and give you another
opportunity to rectify the situation.

## Build an ACK controller

Define the service you want to build and test an ACK controller for by setting
the `SERVICE` environment variable, in our case for Amazon ECR:

```
SERVICE="ecr"
```

Now we are in a position to generate the ACK service controller for the AWS ECR
API and output the generated code to the `services/$SERVICE` directory:

```
make build-controller SERVICE=$SERVICE
```

Above generates the custom resource definition (CRD) manifests for resources
managed by that ACK service controller. It further generates the Helm chart
that can be used to install those CRD manifests, and a deployment manifest 
that runs the ACK service controller in a pod on a Kubernetes cluster (still TODO).

## Run tests

Time to run the tests, so execute:

```
make kind-cluster -s SERVICE=$SERVICE
```

This provisions a Kubernetes cluster using `kind`, builds a container image with
the ACK service controller, and loads the container image into the `kind` cluster.
It then installs the ACK service controller and related Kubernetes manifests into
the `kind` cluster using `kustomize build | kubectl apply -f -`.

Then, the above script runs a series of bash test scripts that call `kubectl`
and the `aws` CLI tools to verify that custom resources of the type managed by
the respective ACK service controller is created, updated and deleted
appropriately (still TODO).

The script deletes the `kind` cluster. You can prevent this last
step from happening by passing the `-p` (for "preserve") flag to the
`scripts/kind-build-test.sh` script or by `make kind-cluster-preserve SERVICE=ecr`.

Finally, if you would like to do functional testing, where you would like to create a AWS Services 
in your account using `KinD` cluster, please use `-r` and pass in `AWS ROLE ARN` to `scripts/kind-build-test.sh` script or
`make kind-cluster-functional SERVICE=ecr ROLE_ARN=arn:aws:iam::12345678980:role/Admin-k8s`

Note: For above functional test, `sts-assume-role.sh` under `scripts/lib` directory will 
make `aws sts assume-role --role-session-arn $AWS_ROLE_ARN --role-session-name $TEMP_ROLE` API call to fetch `AWS_ACCESS_KEY_ID`, 
`AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` variable and inject them as secrets to the deployment. The duration of the session token is 900 seconds (15 minutes)

!!! tip "Tracking testing"
    We track testing in the umbrella [issue 6](https://github.com/aws/aws-controllers-k8s/issues/6).
    on GitHub. Use this issue as a starting point and if you create a new
    testing-related issue, mention it from there.

## Clean up test runs

To clean up a `kind` Kubernetes cluster, which includes all the
configuration files created by the script specifically for your test cluster,
execute:

```
kind delete cluster --name $CLUSTER_NAME
```
or to delete all kind cluster running on your machine execute: 
```
make delete-all-kind-clusters
```