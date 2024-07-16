# Pre-requisites

- [Hugo](https://gohugo.io/installation/)

## How to run locally

```sh
hugo server -D
```

## How to build static site

```sh
hugo
```

### How to run markdown linter locally

```sh
docker run -v $PWD:/workdir ghcr.io/igorshubovych/markdownlint-cli:latest "*.md"
```
