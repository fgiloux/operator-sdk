---
title: Scorecard
weight: 25
---


## Overview

The scorecard command, part of the operator-sdk, executes tests
on your operator based upon a configuration file and test images.

Tests are implemented within test images that are configured
and constructed to be executed by scorecard.

Scorecard assumes it is being executed with access to a configured
Kubernetes cluster.  Each test is executed within a Pod by scorecard,
from which pod logs are aggregated and test results sent to the console.

Scorecard has built-in basic and OLM tests, and it also provides a
means to execute custom test definitions.

## Requirements

The scorecard tests make no assumptions as to the state of the
operator being tested. Creating operators and custom resources
for an operator are left outside the scope of the scorecard itself.

Scorecard tests can however create whatever resources they
require if the tests are designed for resource creation.

## Running the Scorecard

1. Define a scorecard configuration file `config.yaml`.  A sample
configuration file [config.yaml][sample-config] is found within
the SDK repository. See the [config file section](#config-file) for an explaination
of the configuration file format.  Unless you are executing custom
tests, you can just copy the provided example configuration file
into your project.
2. Place the scorecard configuration file within your project
bundle directory at the following location `tests/scorecard/config.yaml`.
You can override the default location of the configuration file
by specifying the `--config` flag.
3. Execute the [`scorecard` command][cli-scorecard]. See the
[command args section](#command-args) for an overview of command invocation.

## Configuration

The scorecard test execution is driven by a configuration file typically named
`config.yaml`. The configuration file is located at the following
location within your bundle:
```
tests/scorecard/config.yaml
```

### Config File

A complete scorecard configuration file can be found [here][sample-config]
and used for running the scorecard pre-defined tests that ship with the SDK.

A sample of the scorecard configuration file may look as follows:

```yaml
stages:
- parallel: true
  tests:
  - image: quay.io/operator-framework/scorecard-test:dev
    entrypoint:
    - scorecard-test
    - basic-check-spec
    labels:
      suite: basic
      test: basic-check-spec-test
  - image: quay.io/operator-framework/scorecard-test:dev
    entrypoint:
    - scorecard-test
    - olm-bundle-validation
    labels:
      suite: olm
      test: olm-bundle-validation-test
```

The configuration file defines the tests that scorecard executes. Tests are
grouped into stages for fine-grained control of [parallelism](#parallelism).
The following fields of the scorecard configuration file define the test as
follows:


| Config Field | Description
| ------------ | -----------
| image        | the test container image name that implements a test
| entrypoint   | the command and arguments that are invoked in the test image to execute a test
| labels       | scorecard-defined or custom labels that [select](#selecting-tests) which tests to run

### Command Args

The scorecard command has the following syntax:
```
operator-sdk scorecard <bundle_dir_or_image> [flags]
```

The scorecard requires a positional argument that holds either the
on-disk path to your operator bundle or the name of a bundle image.

For further information about the flags see the [CLI documentation][cli-scorecard].

## Parallelism

The configuration file allows operator developers to define separate stages for
their tests. Stages run sequentially in the order they are defined in the
configuration file. A stage contains a list of tests and a configurable
`parallel` setting.

By default (or when a stage explicitly sets `parallel` to `false`), tests in
a stage are run sequentially in the order they are defined in the configuration
file. Running tests one at a time is helpful to guarantee that no two tests
interact and conflict with each other.

However, if tests are designed to be fully isolated, they can be parallelized.
To run a set of isolated tests in parallel, include them in the same stage and
set `parallel` to `true`. All tests in a parallel stage are executed
simultaneously, and scorecard waits for all of them to finish before proceding
to the next stage. This can make your tests run much faster.

## Selecting Tests

Tests are selected by setting the `--selector` CLI flag to
a set of label strings.  If a selector flag is not supplied, then all
the tests within the scorecard configuration file are executed.

Tests are executed serially, one after the other, with test results
being aggregated by scorecard and written to stdout.

To select a single test (`basic-check-spec-test`) you would enter the
following:
```sh
$ operator-sdk scorecard -o text --selector=test=basic-check-spec-test
```

To select a suite of tests, olm in this case, you would specify
a label that is used by all the OLM tests:
```sh
$ operator-sdk scorecard -o text --selector=suite=olm
```

To select multiple tests, you could specify them as follows:
```sh
$ operator-sdk scorecard -o text --selector='test in (basic-check-spec-test,olm-bundle-validation-test)'
```

## Built-in Tests

The scorecard ships with pre-defined tests that are arranged into suites.

### Basic Test Suite

| Test        | Description   | Test Name |
| --------    | -------- | -------- |
| Spec Block Exists | This test checks the Custom Resource (CRs) created in the cluster to make sure that all CRs have a spec block. | basic-check-spec-test |

### OLM Test Suite

| Test        | Description   | Short Name |
| --------    | -------- | -------- |
| Bundle Validation | This test validates the bundle manifests found in the bundle that is passed into scorecard.  If the bundle contents contain errors, then the test result output will include the validator log as well as error messages from the validation library.  See this [document][olm-bundle] for details on bundles.| olm-bundle-validation-test |
| Provided APIs have validation |This test verifies that the CRDs for the provided CRs contain a validation section and that there is validation for each spec and status field detected in the CR. | olm-crds-have-validation-test |
| Owned CRDs Have Resources Listed | This test makes sure that the CRDs for each CR provided via the `cr-manifest` option have a `resources` subsection in the [`owned` CRDs section][owned-crds] of the CSV. If the test detects used resources that are not listed in the resources section, it will list them in the suggestions at the end of the test. Users are required to fill out the resources section after initial code generation for this test to pass.  | olm-crds-have-resources-test |
| Spec Fields With Descriptors | This test verifies that every field in the Custom Resources' spec sections have a corresponding descriptor listed in the CSV.| olm-spec-descriptors-test |
| Status Fields With Descriptors | This test verifies that every field in the Custom Resources' status sections have a corresponding descriptor listed in the CSV.| olm-status-descriptors-test |

## Scorecard Output

The `--output` flag specifies the scorecard results output format.

### JSON format

See an example of the JSON format produced by a scorecard test:

```json
{
  "apiVersion": "scorecard.operatorframework.io/v1alpha3",
  "kind": "TestList",
  "items": [
    {
      "kind": "Test",
      "apiVersion": "scorecard.operatorframework.io/v1alpha3",
      "spec": {
        "image": "quay.io/operator-framework/scorecard-test:dev",
        "entrypoint": [
          "scorecard-test",
          "olm-bundle-validation"
        ],
        "labels": {
          "suite": "olm",
          "test": "olm-bundle-validation-test"
        }
      },
      "status": {
        "results": [
          {
            "name": "olm-bundle-validation",
            "log": "time=\"2020-06-10T19:02:49Z\" level=debug msg=\"Found manifests directory\" name=bundle-test\ntime=\"2020-06-10T19:02:49Z\" level=debug msg=\"Found metadata directory\" name=bundle-test\ntime=\"2020-06-10T19:02:49Z\" level=debug msg=\"Getting mediaType info from manifests directory\" name=bundle-test\ntime=\"2020-06-10T19:02:49Z\" level=info msg=\"Found annotations file\" name=bundle-test\ntime=\"2020-06-10T19:02:49Z\" level=info msg=\"Could not find optional dependencies file\" name=bundle-test\n",
            "state": "pass"
          }
        ]
      }
    }
  ]
}
```

### Text format

See an example of the text format produced by a scorecard test:

```
--------------------------------------------------------------------------------
Image:      quay.io/operator-framework/scorecard-test:dev
Entrypoint: [scorecard-test olm-bundle-validation]
Labels:
	"suite":"olm"
	"test":"olm-bundle-validation-test"
Results:
	Name: olm-bundle-validation
	State: pass
	Log:
		time="2020-07-15T03:19:02Z" level=debug msg="Found manifests directory" name=bundle-test
		time="2020-07-15T03:19:02Z" level=debug msg="Found metadata directory" name=bundle-test
		time="2020-07-15T03:19:02Z" level=debug msg="Getting mediaType info from manifests directory" name=bundle-test
		time="2020-07-15T03:19:02Z" level=info msg="Found annotations file" name=bundle-test
		time="2020-07-15T03:19:02Z" level=info msg="Could not find optional dependencies file" name=bundle-test
```

**NOTE** The output format spec for each test matches the [`Test`](https://godoc.org/github.com/operator-framework/operator-sdk/pkg/apis/scorecard/v1alpha3#Test) type layout.


## Exit Status

The scorecard return code is 1 if any of the tests executed did not
pass and 0 if all selected tests pass.

## Extending the Scorecard with Custom Tests

Scorecard will execute custom tests if they follow these mandated conventions:

 * tests are implemented within a container image
 * tests accept an entrypoint which include a command and arguments
 * tests produce v1alpha3 scorecard output in JSON format with no extraneous logging in the test output
 * tests can obtain the bundle contents at a shared mount point of /bundle
 * tests can access the Kubernetes API using an in-cluster client connection

See [here][custom-image] for an example of a custom test image written in golang.

Writing custom tests in other programming languages is possible
if the test image follows the above guidelines.


[cli-scorecard]: /docs/cli/golang/operator-sdk_scorecard/
[sample-config]: https://github.com/operator-framework/operator-sdk/blob/master/internal/scorecard/testdata/bundle/tests/scorecard/config.yaml
[custom-image]: https://github.com/operator-framework/operator-sdk/blob/master/internal/scorecard/examples/custom-scorecard-tests
[scorecard-struct]: https://github.com/operator-framework/operator-sdk/blob/master/pkg/apis/scorecard/v1alpha3/types.go
[olm-bundle]:https://github.com/operator-framework/operator-registry#manifest-format