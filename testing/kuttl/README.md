# Kuttl

## Installing
Docs for install: https://kuttl.dev/docs/cli.html#setup-the-kuttl-kubectl-plugin

Options:
  - Download and install the binary
  - Install the `kubectl krew` [plugin manager](https://github.com/kubernetes-sigs/krew)
    and `kubectl krew install kuttl`
## Cheat sheet

### Suppressing Noisy Logs

KUTTL gives you the option to suppress events from the test logging output. To enable this feature
update the `kuttl` parameter when calling the `make` target

```
KUTTL_TEST='kuttl test --suppress-log=events' make check-kuttl
```

To suppress the events permanently, you can add the following to the KUTTL config (kuttl-test.yaml)
```
suppress: 
- events
```

### Run test suite

Make sure that the operator is running in your kubernetes environment and that your `kubeconfig` is
set up. Then run the make target:

```
make check-kuttl
```

### Running a single test
A single test is considered to be one directory under `kuttl/e2e`, for example
`kuttl/e2e/restore` would run the `restore` test.

There are two ways to run a single test in isolation: 
- using an env var with the make target: `KUTTL_TEST='kuttl test --test <test-name>' make check-kuttl`
- using `kubectl kuttl --test` flag: `kubectl kuttl test testing/kuttl/e2e --test <test-name>`

### Writing additional tests

To make it easier to read tests, we want to put our `assert.yaml`/`errors.yaml` files after the
files that create/update the objects for a step. To achieve this, infix an extra `-` between the
step number and the object/step name.

For example, if the `00` test step wants to create a cluster and then assert that the cluster is ready,
the files would be named

```console
00--cluster.yaml # note the extra `-` to ensure that it sorts above the following file
00-assert.yaml
```

### Generating tests

KUTTL is good at setting up K8s objects for testing, but does not have a native way to dynamically
change those K8s objects before applying them. That means that, if we wanted to write a cluster
connection test for PG 13 and PG 14, we would end up writing two nearly identical tests.

Rather than write those multiple tests, we are using `envsubst` to replace some common variables
in a `source` template YAML and writing those files to the `testing/kuttl/generated` folder.

These templated test files can be generated by setting some variables in the command line and
calling the `make generate-kuttl` target:

```console
KUTTL_PG_VERSION=13 KUTTL_PSQL_IMAGE=registry.developers.crunchydata.com/crunchydata/crunchy-postgres:centos8-13.5-0 make generate-kuttl
```

This will loop through the folders in the `source` folder and create matching folders in the
`generated` folder that can be checked for correctness before running the tests. (The files in
the `generated` folder will not be checked into github; our CI runn will generate and test the
files from scratch.)

For ease of use, the `check-generated-kuttl` make target will both generate those tests and test
them, again with the set environment variables:

```console
KUTTL_PG_VERSION=13 KUTTL_PSQL_IMAGE=registry.developers.crunchydata.com/crunchydata/crunchy-postgres:centos8-13.5-0 make check-generated-kuttl
```

To prevent errors, we want to set defaults for all the environment variables used in the `source`
YAML files; so if you add a new test with a new variable, please update the Makefile with a
reasonable/preferred default.