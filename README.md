# drone-lambda

[![GoDoc](https://godoc.org/github.com/appleboy/drone-lambda?status.svg)](https://godoc.org/github.com/appleboy/drone-lambda)
[![codecov](https://codecov.io/gh/appleboy/drone-lambda/branch/master/graph/badge.svg)](https://codecov.io/gh/appleboy/drone-lambda)
[![Go Report Card](https://goreportcard.com/badge/github.com/appleboy/drone-lambda)](https://goreportcard.com/report/github.com/appleboy/drone-lambda)
[![Docker Pulls](https://img.shields.io/docker/pulls/appleboy/drone-lambda.svg)](https://hub.docker.com/r/appleboy/drone-lambda/)

Deploying Lambda code with drone CI to an existing function. The plugin automatically deployes a serverless function to AWS Lambda from a zip file located in an S3 bucket. This plugin does not handle creating or uploading the zip file.

![cover](./images/infra.svg)

## Build or Download a binary

The pre-compiled binaries can be downloaded from [release page](https://github.com/appleboy/drone-lambda/releases). Support the following OS type.

* Windows amd64/386
* Linux amd64/386
* Darwin amd64/386

With `Go` installed

```bash
go install github.com/appleboy/drone-lambda@latest
```

or build the binary with the following command:

```sh
export GOOS=linux
export GOARCH=amd64
export CGO_ENABLED=0
export GO111MODULE=on

go test -cover ./...

go build -v -a -tags netgo -o release/linux/amd64/drone-lambda .
```

## Docker

Build the docker image with the following commands:

```bash
make docker
```

## Usage

There are three ways to send notification.

### Usage from binary

Update lambda function from zip file.

```sh
$ drone-lambda --region ap-southeast-1 \
  --access-key xxxx \
  --secret-key xxxx \
  --function-name upload-s3 \
  --zip-file deployment.zip
```

Update lambda function from s3 object.

```sh
$ drone-lambda --region ap-southeast-1 \
  --access-key xxxx \
  --secret-key xxxx \
  --function-name upload-s3 \
  --s3-bucket some-bucket \
  --s3-key lambda-dir/lambda-project-${DRONE_BUILD_NUMBER}.zip
```

### Usage from docker

Update lambda function from zip file.

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=xxxxxxx \
  -e AWS_SECRET_ACCESS_KEY=xxxxxxx \
  -e FUNCTION_NAME=upload-s3 \
  -e ZIP_FILE=deployment.zip \
  -v $(pwd):$(pwd) \
  -w $(pwd) \
  appleboy/drone-lambda
```

Update lambda function from s3 object.

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=xxxxxxx \
  -e AWS_SECRET_ACCESS_KEY=xxxxxxx \
  -e FUNCTION_NAME=upload-s3 \
  -e S3_BUCKET=some-bucket \
  -e S3_KEY=lambda-project.zip \
  appleboy/drone-lambda
```

### Usage from drone ci

Update lambda function, execute from the working directory:

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=xxxxxxx \
  -e AWS_SECRET_ACCESS_KEY=xxxxxxx \
  -e FUNCTION_NAME=upload-s3 \
  -e ZIP_FILE=deployment.zip \
  -v $(pwd):$(pwd) \
  -w $(pwd) \
  appleboy/drone-lambda
```

## Deploy with Drone

```yaml
kind: pipeline
name: default

steps:
- name: build
  image: golang:1.15
  commands:
  - apt-get update && apt-get -y install zip
  - cd example && GOOS=linux go build -v -a -o main main.go && zip deployment.zip main

- name: deploy-lambda
  image: appleboy/drone-lambda
  settings:
    pull: true
    access_key:
      from_secret: AWS_ACCESS_KEY_ID
    secret_key:
      from_secret: AWS_SECRET_ACCESS_KEY
    region:
      from_secret: AWS_REGION
    function_name: gorush
    zip_file: example/deployment.zip
    debug: true
```

## AWS Policy

Add the following AWS policy if you want to integrate with CI/CD tools like Jenkins, GitLab Ci or Drone. Your function needs permission to upload trace data to [X-Ray](https://docs.aws.amazon.com/lambda/latest/dg/services-xray.html). When you activate tracing in the Lambda console, Lambda adds the required permissions to your function's execution role. Otherwise, add the [AWSXRayDaemonWriteAccess](https://console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess) policy to the execution role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "iam:ListRoles",
        "lambda:UpdateFunctionCode",
        "lambda:CreateFunction",
        "lambda:GetFunction",
        "lambda:GetFunctionConfiguration",
        "lambda:UpdateFunctionConfiguration"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords",
        "xray:GetSamplingRules",
        "xray:GetSamplingTargets",
        "xray:GetSamplingStatisticSummaries"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```
