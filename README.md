# Adding Container Security and Compliance Scanning to your AWS CodeBuild pipeline.

## Introduction

This will walkthrough integrating Anchore scanning with AWS CodeBuild. During the first step, a Docker image will be built from a Dockerfile. Following this, during the second step, Anchore will scan the image, and depending on the result of the policy evaluation, proceed to the final step. During the final step the built image will be pushed to a Docker registry.

## Prerequisites

- Running Anchore Engine service
- AWS account
- Repository that contains a Dockerfile

## Setup

Prior to setting up your AWS CodeBuild pipeline, an Anchore Engine service needs to be accessible from the pipeline. Typically this is on port 8228. In this example, I have an Anchore Engine service on AWS EC2 with standard configuration. I also have a Dockerfile in a GitHub repository that I will build an image from during the first step of the pipeline. In the final step, I will be pushing the built image to an image repository in my personal Dockerhub.

The GitHub repository can be referenced [here](https://github.com/valancej/aws-codepipeline-anchore-demo).

I've added the following environment variables in the build project setup:

- ANCHORE_CLI_URL
- ANCHORE_CLI_USER
- ANCHORE_CLI_PASS
- ANCHORE_CLI_FAIL_ON_POLICY
- dockerhubUser
- dockerhubPass

A `buildspec.yml` file should exist in the root directory of the Github repository you will link to your CodeBuild setup.

## Install

In the install phase of the buildspec.yml file we install the Anchore CLI. You can find more info by referencing the GitHub repo [here](https://github.com/anchore/anchore-cli).

## Build image

In the build phase of the buildspec.yml file we build and push a Docker image to Docker Hub.

```
build:
    commands:
      - docker build -t jvalance/node_critical_fail .
      - docker push jvalance/node_critical_fail
```

## Conduct image scan

In the post_build phase of the buildspec.yml file we scan the built image with Anchore, and conduct a policy evaluation on it. Depending on the result of the policy evaluation the pipeline may or may not fail. In this example, the evaluation will not be successful, and the built image will not be pushed to a Docker registry. 

```
post_build:
    commands:
      - anchore-cli image add jvalance/node_critical_fail:latest
      - echo "Waiting for image to finish analysis"
      - anchore-cli image wait jvalance/node_critical_fail:latest
      - echo "Analysis complete"
      - if [ "$ANCHORE_FAIL_ON_POLICY" = "true" ] ; then anchore-cli evaluate check jvalance/node_critical_fail:latest  ; fi
      - echo "Pushing image to Docker Hub"
      - docker push jvalance/node_critical_fail
```

As a reminder, we advise having separate Docker registries for images that are being scanned with Anchore, and images that have passed an Anchore scan. For example, a registry for dev/test images, and a registry to certified, trusted, production-ready images. You may have noticed during this walkthrough I am using the same Docker Hub repository for all steps. This is not recommended for a production-grade deployment. 

You can read more about Anchore on our [website](https://anchore.com). Additionally, you can find out more information on AWS CodeBuild by referencing their [documentation](https://docs.aws.amazon.com/codebuild/index.html#lang/en_us).




