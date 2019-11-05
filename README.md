Kustomize test repo. This is a 12 factor nginx app, configurable via env vars:

- TITLE: H1 at the top of the page
- COLOR: bgcolor of the whole page (lightgreen/lightblue/...)
- BODY: additional html under H1

## Usage

Create a NodePort type service
```
k apply -k https://github.com/lalyos/kustom-lunch/base
```

## With Ingress

Create it with an additional **ingress**. Try an interactive create with on-the-fly domain modification:
```
k create --edit -k https://github.com/lalyos/kustom-lunch/with-ing
```
This demonstrates that online kustomization.yaml handles correctly even
bases references with relative references:
```
bases:
  - ../base
```

## Exercices

See [EXERCISES.md](EXERCISES.md) about proper environment variable handling.