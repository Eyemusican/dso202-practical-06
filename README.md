# DSO202 Practical 6: Helm

This is my work for Practical 6, following the Unit III 3.2 Helm notes. I used Helm 4 to consume a published chart (podinfo), then built my own chart called webapp for an nginx page, configured it for dev and prod with values files, and validated it with lint, template, a values schema and server dry runs.

## Note on the instructions

The lab page (https://hackmd.io/RwJPCvlnR727QEgZI_zQYg) ends at section 3.2.3.2. The notes mention Demo Stages 6, 7 and 8 and sections 3.2.4 to 3.2.8 (hooks and tests, dependencies, CRDs, distribution), but that content was not published on the page when I did this. So this repo covers Demo Stages 0 to 5, which is everything the page actually instructs. Section 3.2.3 has no commands, so I covered it in the report.

## Issues I ran into

- The podinfo image took too long to pull inside the kind nodes, so the first install timed out after 3 minutes and was marked failed. I pulled the image on my laptop with docker, loaded it into the cluster with kind load, uninstalled the failed release and installed again cleanly.
- My kubectl context still pointed at the dso202-assignment-01 namespace from Assignment 1, so HELM_NAMESPACE was not default. I reset it to default to match the notes.
- My Assignment 1 frontend Service was still holding NodePort 30080, which the dev values use. I deleted that Service so the port was free.
- My checksum annotation is different from the one in the notes because the hash depends on the exact bytes of the rendered ConfigMap. I showed it is repeatable by rendering 3 times and getting the same hash.

## Versions used

- kind cluster, 1 control plane, 2 workers, Kubernetes v1.36.1
- Helm v4.3.0
- podinfo chart 6.15.0
- nginx 1.30-alpine

## Repo structure

```
dso202-practical-06/
├── README.md
├── webapp/
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values.schema.json
│   ├── .helmignore
│   ├── charts/
│   └── templates/
│       ├── _helpers.tpl
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── NOTES.txt
│       └── tests/
│           └── test-connection.yaml
├── environments/
│   ├── dev.yaml
│   └── prod.yaml
├── evidence/
└── report/
```

## How to check the chart

```bash
for env in dev prod; do
  helm lint ./webapp -f environments/$env.yaml &&
  helm template webapp-$env ./webapp -f environments/$env.yaml > /dev/null &&
  echo "$env: ok"
done
```

## How to install it

```bash
helm install webapp-dev ./webapp -f environments/dev.yaml -n dso202-dev --create-namespace
curl http://localhost:30080

helm install webapp-prod ./webapp -f environments/prod.yaml -n dso202-prod --create-namespace
```

Each environment file is used on its own on top of the chart defaults. Layering dev.yaml and prod.yaml together leaks settings from one into the other, which I showed in Stage 4.

## Cleanup

```bash
helm uninstall webapp-dev -n dso202-dev
helm uninstall webapp-prod -n dso202-prod
kubectl delete namespace dso202-helm dso202-dev dso202-prod
```

Uninstall removes every object in the release and its revision Secrets, but not the namespace, since creating it with --create-namespace does not make it part of the release.

## Report

Full stage by stage writeup with screenshots and answers to the review questions is in report/practical-06-report.md.