# GitHub Action to install and setup [`run-job`](https://github.com/alexellis/run-job)

[![Build](https://github.com/StevenACoffman/setup-run-job/actions/workflows/use-action.yaml/badge.svg)](https://github.com/StevenACoffman/setup-run-job/actions/workflows/use-action.yaml)


This action will just install [run-job](https://github.com/alexellis/run-job). 

### What is run-job?
[run-job](https://github.com/alexellis/run-job) is a brilliant tool to run a Kubernetes Job and get the logs when it's done 🏃‍.

## Example usage

```yaml
name: Run Job for District
on:
  workflow_dispatch:
    inputs:
      district_id:
        description: District Key ID
        required: true
env:
  PROJECT_ID: XXXXXX
  GKE_CLUSTER: XXXXX
  GKE_ZONE: us-central1
jobs:
  provision:
    #    runs-on: [self-hosted, linux, x64]
    runs-on: ubuntu-latest
    timeout-minutes: 30
    # Adds "id-token" with the intended permissions.
    permissions:
      contents: 'read'
      id-token: 'write'
    steps:
      - name: Clone Repository (Latest)
        uses: actions/checkout@v4
      - name: Install Go
        uses: actions/setup-go@v5
        with:
          go-version: 1.21.x
          cache-dependency-path: "**/*.sum"
      #    - name: Setup kubectl
      #      uses: azure/setup-kubectl@v3.2
      - name: Install KO
        uses: imjasonh/setup-ko@v0.6
      - name: Install Run Job
        uses: StevenACoffman/setup-run-job@v0.0.4
      - name: 'Authenticate to Google Cloud'
        id: 'auth'
        uses: 'google-github-actions/auth@v2'
        with:
          workload_identity_provider: 'projects/XXXXXX/locations/global/workloadIdentityPools/github-action-pool/providers/github-action-provider'
          service_account: 'XXXXXX@XXXXXX.iam.gserviceaccount.com'
          #  token_format: 'access_token'
      - id: 'get-kube-credentials'
        name: 'get kubernetes credentials'
        uses: 'google-github-actions/get-gke-credentials@v2'
        with:
          cluster_name: 'mycluster'
          location: 'us-central1'
      - id: 'build-image-deploy-kube-and-update-manifests'
        name: 'build-image-deploy-kube-and-update-manifests'
        run: |-
          # Build docker image without docker, tag image as Git commit SHA1
          export KO_DOCKER_REPO=us-central1-docker.pkg.dev/myproject/myregistry/myapp
          export GOPRIVATE=github.com/MyOrg
          export GO111MODULE=on
          export COMMIT_SHA=$(git rev-parse HEAD)
          ko publish --sbom=none --bare --platform=linux/amd64 . -t ${COMMIT_SHA}
          # Create run-job Kubernetes template file with docker tag
          echo "name: runjob" > runjob.yaml
          echo "namespace: jobber" >> runjob.yaml
          echo "image: ${KO_DOCKER_REPO}:$(git rev-parse HEAD)" >> runjob.yaml
          echo "args:" >> runjob.yaml
          echo "  - publish" >> runjob.yaml
          echo "  - -district=${{ github.event.inputs.app }}" >> runjob.yaml
          # run run-job
          run-job -kubeconfig "${KUBECONFIG}" -f runjob.yaml
```

_That's it!_ This workflow will install run-job so you can use it with ko or docker (or whatever).

You probably want to [set up Workload Identity](https://github.com/google-github-actions/auth#usage) between your GitHub Actions workflow and your GCP project.

### Select `run-job` version to install

By default, `setup-run-job` installs the [latest released version of `run-job`](https://github.com/alexellis/run-job/releases).

You can select a run-job release version with the `version` parameter:

```yaml
- uses: StevenACoffman/setup-run-job@v0.0.4
  with:
    version: 0.1.1
```

To build and install `run-job` from source using `go install`, specify `version: tip`.

```yaml
steps:
...
- uses: StevenACoffman/setup-run-job@v0.0.4
- name: 'Run a Job in Kubernetes'
  run: |
    run-job -kubeconfig "${KUBECONFIG}" -f runjob.yaml
```


