ci-operator
===========

This document describes how to create CI jobs for Openshift components using
ci-operator and is intended for component developers who want to add tests to
their CI process.


End-to-end tests
----------------

This section describes how to configure end-to-end tests using ci-operator.  In
this context, "end-to-end" means the functionality of the application is being
tested on top of a Kubernetes cluster from an end-user perspective.

The preferred way to write this type of tests is using `ci-operator`.  See the
documentation for details on how to download, build, and execute it:

https://github.com/openshift/ci-operator.git

ci-operator requires a configuration file for the repository being tested.
These files are located in the [`config`](config/) directory.  The ci-operator
repository has documentation for adding a new configuration file in case one
doesn't already exist.  These files take care of most of the CI process:
downloading the source, building binaries, building RPMs, creating images,
executing unit tests, etc., and can be built upon for e2e tests with little or
no modification.

To add an e2e test:

1. Determine the pre-requisites for the test.  Practically, this means choosing
   the ci-operator template according to the type of test.  There are already
   files in the [`templates`](templates/) directory for the most common cases,
   see [Using a template](#using-a-template) below.
2. Determine and configure the template's inputs.  This is specific to each
   template and should be documented in its parameters.  The e2e test might
   need minor modifications to fit the environment created by the template.
3. Add one or more Prow jobs to Prow's configuration file with the information
   gathered in the previous steps.


# Provisioning a cluster

Contrary to other types of tests, e2e tests usually require a cluster, not just
a single container.  While there aren't yet native primitives in `ci-operator`
for cluster provisioning, it provides one open-ended feature that can be
leveraged to accomplish that: template steps.

Template steps allow the creation of arbitrary objects in the cluster where the
CI pipeline is executed.  This is used to start a pod that will then provision
a separate cluster for the tests.  This directory already contains a few
templates that can be used either directly or (rarely necessary, in practice)
as a reference:

- `cluster-launch-e2e.yaml`: launches a cluster in GCP using openshift-ansible
  and runs Origin e2e tests on it, parameterized by test focus.
- `cluster-launch-installer-e2e.yaml`: same as `cluster-launch-e2e.yaml`, but
  uses `openshift-installer` instead of `openshift-ansible`.
- `cluster-launch-src.yaml`: launches a cluster in GCP using openshift-ansible
  and runs a script from the repository being tested with the resulting
  `$KUBECONFIG`, parameterized by test script.
- `master-sidecar.yaml`: spins up a simple openshift control plane as a sidecar
  and waits for the `COMMAND` specified to the template to be executed, before
  itself exiting. The test container is given access to the generated
  configuration and the `admin.kubeconfig`.


# Using a template

Normally, ci-operator executes all steps defined in its configuration file or,
with the `--target` argument, only a single step and its dependencies.  A
template can be added as a step at runtime with the `--template` option and can
also be used as a target.  The step is named after the name of the template, if
it has one, or the `basename` of the file path passed to the `--template`
option, without the extension.

Several parameters are provided by ci-operator when it instantiates the
template that contain information about the test: the job name, which namespace
it's operating into, URLs for images, etc.  Run `ci-operator --help` for a
comprehensive list of parameters that can be supplied to a template.

When artifact collection is enabled (using the `--artifacts-dir
/path/to/artifacts` option, see the ci-operator documentation linked at the top
of this document for details), logs from pods that fail will be fetched and
stored.  If part of the output is necessary even in case of success, it can be
written to the artifact directory.

Below is a sample job extracted from the
[jobs directory](jobs/openshift/descheduler/) with comments added to describe
the relevant sections.  For a complete description of the fields, see prow's
documentation:

https://github.com/kubernetes/test-infra/tree/master/prow

This job uses the `cluster-launch-src.yaml` template to provision a GCP cluster
and run an e2e test script.  Note that there currently is a lot of unnecessary
complexity and duplication, which should be cleared in the future.


```yaml
- name: pull-ci-openshift-descheduler-e2e-gce-3.10
  agent: kubernetes
  always_run: true
  context: ci/prow/e2e
  # each branch needs its own ci-operator config
  branches:
  - release-3.10
  rerun_command: "/test e2e"
  trigger: "((?m)^/test( all| e2e),?(\\s+|$))"
  decorate: true
  skip_cloning: true
  spec:
    serviceAccountName: ci-operator
    volumes:
    # ci-operator template, loaded into a configmap in the ci cluster from the
    # files in ci-operator/templates
    - name: job-definition
      configMap:
        name: prow-job-cluster-launch-src
    # configuration for gcp clusters, shared by all jobs
    - name: cluster-profile
      projected:
        sources:
        - secret:
            name: cluster-secrets-gcp
        - configMap:
            name: cluster-profile-gcp
    # the actual ci-operator container
    containers:
    - name: test
      image: ci-operator:latest
      volumeMounts:
      - name: job-definition
        mountPath: /usr/local/e2e-gcp
        subPath: cluster-launch-src.yaml
      - name: cluster-profile
        mountPath: /usr/local/openshift-descheduler-e2e-cluster-profile
      env:
      # unique identifier: names gcp instances, etc.
      - name: JOB_NAME_SAFE
        value: openshift-descheduler-e2e
      - name: CLUSTER_TYPE
        value: gcp
      # ci-operator configuration file, loaded into a configmap in the cluster
      # from the files in ci-operator/config
      - name: CONFIG_SPEC
        valueFrom:
          configMapKeyRef:
            name: ci-operator-openshift-descheduler
            key: release-3.10.json
      # the actual test command (the variable name and format is specific to
      # the template being used (cluster-launch-src.yaml, in this case), see
      # the definition for more details)
      - name: TEST_COMMAND
        value: "cp /tmp/admin.kubeconfig /tmp/admin.conf && make test-e2e"
      # required for jobs that use openshift-ansible to create a cluster
      - name: RPM_REPO_BASEURL_REF
        value: https://storage.googleapis.com/origin-ci-test/releases/openshift/origin/release-3.10/.latest-rpms
      # actual ci-operator call
      command:
      - /bin/bash
      - -c
      - |
        #!/bin/bash
        set -e
        export RPM_REPO="$(curl -q "${RPM_REPO_BASEURL_REF}" 2>/dev/null)"
        ci-operator \
          --artifact-dir=$(ARTIFACTS) \
          --secret-dir=/usr/local/openshift-descheduler-e2e-cluster-profile \
          --template=/usr/local/e2e-gcp \
          --target=e2e-gcp
```