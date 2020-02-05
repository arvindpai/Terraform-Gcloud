# Terraform-Gcloud

This repository explains how to build a terraform docker which includes gcloud and execute a terraform script within cloud build.


## Execution Points:

### /cloudbuild-docker

In order to execure this execution point - gcloud builds submit --config=docker.yaml

#### /cloudbuild-docker/Dockerfile

The following Dockerfile contains terraform along with gcloud sdk. As a part of the gcloud sdk kubectl is installed. For this reason kubectl does not need to be installed separately.

```
FROM alpine:3.9

ARG TERRAFORM_VERSION
ARG TERRAFORM_VERSION_SHA256SUM

COPY terraform_${TERRAFORM_VERSION}_linux_amd64.zip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
RUN echo "${TERRAFORM_VERSION_SHA256SUM}  terraform_${TERRAFORM_VERSION}_linux_amd64.zip" > checksum && sha256sum -c checksum
RUN unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip

FROM gcr.io/cloud-builders/gcloud

COPY --from=0 terraform /usr/bin/terraform
COPY entrypoint.bash /builder/entrypoint.bash
ENTRYPOINT ["/builder/entrypoint.bash"]


```

#### /cloudbuild-docker/docker.yaml

The following builds a docker and pushes it to continer repository. 

gcr.io in the tag section is required in order to push the image to container registery.
```
# This builds a docker with gcloud installed
substitutions:
  _TERRAFORM_VERSION: 0.12.9
  _TERRAFORM_VERSION_SHA256SUM: 69712c6216cc09b7eca514b9fb137d4b1fead76559c66f338b4185e1c347ace5
steps:
- name: 'gcr.io/cloud-builders/wget'
  args: ["https://releases.hashicorp.com/terraform/${_TERRAFORM_VERSION}/terraform_${_TERRAFORM_VERSION}_linux_amd64.zip"]
- name: 'gcr.io/cloud-builders/docker'
  env:
  - 'TERRAFORM_VERSION=${_TERRAFORM_VERSION}'
  - 'TERRAFORM_VERSION_SHA256SUM=${_TERRAFORM_VERSION_SHA256SUM}'
  args:
  - build
  - --build-arg
  - TERRAFORM_VERSION=${_TERRAFORM_VERSION}
  - --build-arg
  - TERRAFORM_VERSION_SHA256SUM=${_TERRAFORM_VERSION_SHA256SUM}
  - --tag
  - gcr.io/${PROJECT_ID}/terraform:${_TERRAFORM_VERSION}
  - --tag
  - gcr.io/${PROJECT_ID}/terraform:latest
  - .
- name: 'gcr.io/${PROJECT_ID}/terraform:${_TERRAFORM_VERSION}'
  args: ['version']
images:
  - 'gcr.io/${PROJECT_ID}/terraform:${_TERRAFORM_VERSION}'
  - 'gcr.io/${PROJECT_ID}/terraform:latest'
tags: ['cloud-builders-community']

  ```
  
####  ./terraform.yaml
In order to execure this execution point - gcloud builds submit --config=terraform.yaml

  
  The following uses the the docker image within the container registry to execute terraform. 
  ```
  - name: 'terraform-gcloud'
  args: ['init']
  env:
    - "TF_VAR_project-name=${PROJECT_ID}"
- name: 'terraform-gcloud'
  args: ['plan']
  env:
    - "TF_VAR_project-name=${PROJECT_ID}"
- name: 'terraform-gcloud'
  args: ['apply', '-auto-approve']
  env:
    - "TF_VAR_project-name=${PROJECT_ID}"
- name: 'terraform-gcloud'
  args: ['destroy', '-auto-approve']
  env:
    - "TF_VAR_project-name=${PROJECT_ID}"
```
