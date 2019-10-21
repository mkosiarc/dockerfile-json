# dockerfile-json

Prints Dockerfiles as JSON to stdout, optionally evaluates build args. Uses the [official Dockerfile parser](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/) from buildkit. Plays well with `jq`.

## Contents

- [Contents](#contents)
- [Get it](#get-it)
- [Usage](#usage)
- [Examples](#examples)
  - [JSON output](#json-output)
  - [Extract build stage names](#extract-build-stage-names)
  - [Extract base images](#extract-base-images)
    - [Expand build args, omit stage aliases and `scratch`](#expand-build-args-omit-stage-aliases-and-scratch)
    - [Set build args, omit stage aliases and `scratch`](#set-build-args-omit-stage-aliases-and-scratch)
    - [Expand build args, include all base names](#expand-build-args-include-all-base-names)
    - [Ignore build args, include all base names](#ignore-build-args-include-all-base-names)

## Get it

Using go get:

```bash
go get -u github.com/keilerkonzept/dockerfile-json
```

Or [download the binary for your platform](https://github.com/keilerkonzept/dockerfile-json/releases/latest) from the releases page.

## Usage

### CLI

```text
dockerfile-json

Usage of dockerfile-json:
  -build-arg value
    	a key/value pair KEY[=VALUE]
  -expand-build-args
    	expand build args (default true)
  -quiet
    	suppress log output (stderr)
```

## Examples

### JSON output

`Dockerfile`
```Dockerfile
ARG ALPINE_TAG=3.10

FROM alpine:$ALPINE_TAG AS build
RUN echo "Hello world" > abc

FROM build AS test
RUN echo "foo" > bar

FROM scratch
COPY --from=build --chown=nobody:nobody abc .
CMD ["echo"]
```

```sh
$ dockerfile-json Dockerfile | jq .
```
```json
{
  "MetaArgs": [
    {
      "Key": "ALPINE_TAG",
      "Value": "3.10"
    }
  ],
  "Stages": [
    {
      "Name": "build",
      "BaseName": "alpine:3.10",
      "SourceCode": "FROM alpine:$ALPINE_TAG AS build",
      "FromStage": false,
      "FromScratch": false,
      "Commands": [
        {
          "Name": "run",
          "Command": {
            "CmdLine": [
              "echo \"Hello world\" > abc"
            ],
            "PrependShell": true
          }
        }
      ]
    },
    {
      "Name": "test",
      "BaseName": "build",
      "SourceCode": "FROM build AS test",
      "FromStage": true,
      "FromStageIndex": 0,
      "FromScratch": false,
      "Commands": [
        {
          "Name": "run",
          "Command": {
            "CmdLine": [
              "echo \"foo\" > bar"
            ],
            "PrependShell": true
          }
        }
      ]
    },
    {
      "BaseName": "scratch",
      "SourceCode": "FROM scratch",
      "FromStage": false,
      "FromScratch": true,
      "Commands": [
        {
          "Name": "copy",
          "Command": {
            "SourcesAndDest": [
              "abc",
              "."
            ],
            "From": "",
            "Chown": "nobody:nobody"
          }
        },
        {
          "Name": "cmd",
          "Command": {
            "CmdLine": [
              "echo"
            ],
            "PrependShell": false
          }
        }
      ]
    }
  ]
}
```

### Extract build stage names

`Dockerfile`
```Dockerfile
FROM maven:alpine AS build
# ...

FROM build AS test
# ...

FROM openjdk:jre-alpine
# ...
```

```sh
$ dockerfile-json Dockerfile | jq '.Stages[] | .Name | select(. != "")'
```
```json
"build"
"test"
```

### Extract base images

`Dockerfile`
```Dockerfile
ARG ALPINE_TAG=3.10
ARG APP_BASE=scratch

FROM alpine:$ALPINE_TAG AS build
# ...

FROM build
# ...

FROM $APP_BASE
# ...
```

#### Expand build args, omit stage aliases and `scratch`

```sh
$ dockerfile-json Dockerfile |
    jq '.Stages[] | select((.FromStage or .FromScratch)|not) | .BaseName'
```
```json
"alpine:3.10"
```

#### Set build args, omit stage aliases and `scratch`

```sh
$ dockerfile-json --build-arg ALPINE_TAG=hello-world Dockerfile |
    jq '.Stages[] | select((.FromStage or .FromScratch)|not) | .BaseName'
```
```json
"alpine:hello-world"
```

#### Expand build args, include all base names

```sh
$  dockerfile-json Dockerfile | jq '.Stages[] | .BaseName'
```
```json
"alpine:3.10"
"build"
"scratch"
```

#### Ignore build args, include all base names

```sh
$ dockerfile-json --expand-build-args=false Dockerfile | jq '.Stages[] | .BaseName'
```
```json
"alpine:$ALPINE_TAG"
"build"
"$APP_BASE"
```
