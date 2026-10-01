# CGI Italy - Insula Processing Core

The open source processing core of [Insula](https://insula.earth),
[CGI's Earth Observation platform](https://www.cgi.com/en/solutions/space/insula-cgi-earth-observation-platform-as-a-service).
It runs Earth Observation applications, packaged as
[OGC Application Packages](https://docs.ogc.org/bp/20-089r1.html) (CWL), on Kubernetes via Argo Workflows
and exposes them through the [OGC API - Processes](https://ogcapi.ogc.org/processes/)
standard.

Insula Processing Core is the processing building block CGI contributes to
[EOEPCA+](https://eoepca.org), the Earth Observation Exploitation Platform Common
Architecture. See [Scope](#scope) for details.

## Repositories

| Repository | What it is |
|------------|------------|
| [`insula`](https://github.com/cgi-italy-insula-processing/insula) | Processing backend: job orchestration server, worker, input downloader, output uploader, Kubernetes event collector and the messaging layer between them. Java 8, Spring Boot 2.7, Gradle. |
| [`ogc-api`](https://github.com/cgi-italy-insula-processing/ogc-api) | OGC API - Processes service, the only public entry point. Server stubs are generated from a trimmed copy of the OGC specification (the README lists every difference); requests are translated into calls to the processing backend. Java 17, Spring Boot 3, Gradle. |
| [`helm-chart`](https://github.com/cgi-italy-insula-processing/helm-chart) | Helm umbrella chart that deploys the whole building block: OGC API, server, worker, Argo Workflows, ActiveMQ, PostgreSQL, MinIO and a container registry. |
| [`deployment`](https://github.com/cgi-italy-insula-processing/deployment) | Bash scripts that check the cluster prerequisites, generate a values file, install the chart and validate the running release. They follow the conventions of the [EOEPCA deployment guide](https://deployment-guide.docs.eoepca.org/current/building-blocks/oapip-engine/). |
| [`minio`](https://github.com/cgi-italy-insula-processing/minio) | Documentation for `ghcr.io/cgi-italy-insula-processing/minio`, an unofficial, unmodified mirror of the upstream MinIO `RELEASE.2021-02-14T04-01-33Z` image that the chart deploys. We republish it because the upstream channels no longer serve this release. The repository holds the provenance evidence, a verification script, the license and notice files and the redistribution sign-off. The image is unpatched legacy software (CVE-2023-28432) and is not affiliated with MinIO, Inc. |

> **Publication in progress.** `helm-chart` and `deployment` are still being published
> and may be empty when you visit. Container images are not yet publicly available;
> they will be published to the GitHub Container Registry under
> `ghcr.io/cgi-italy-insula-processing`. The MinIO image used by the chart is documented in
> [`minio`](https://github.com/cgi-italy-insula-processing/minio).

## How it works

1. **Deploy a process.** `POST /ogcapi/processes` with an OGC Application Package that
   references your CWL. The process then appears under `GET /processes`.
2. **Execute it.** `POST /ogcapi/processes/{processId}/execution` creates an asynchronous job.
   The server validates the inputs, records the job in PostgreSQL and queues it on
   ActiveMQ.
3. **Run it.** The worker takes the job and submits an Argo Workflow in a dedicated
   workflows namespace: the input downloader stages the inputs, your processor
   container runs, and the output uploader stores the results in MinIO (S3-compatible).
4. **Track it.** The Kubernetes event collector watches the workflow pods and reports
   progress back. Poll `GET /ogcapi/jobs/{jobId}`, then fetch `GET /ogcapi/jobs/{jobId}/results`.

## Get started

**Prerequisites.** A Kubernetes cluster with an ingress controller and a storage class
for ReadWriteOnce volumes (about 91Gi with the default sizes), plus `kubectl`, `helm`
3.5 or later, `curl` and Bash 4.4 or later on your machine. cert-manager is optional
(TLS). Processor images must be pullable without credentials.

```bash
git clone https://github.com/cgi-italy-insula-processing/helm-chart.git
git clone https://github.com/cgi-italy-insula-processing/deployment.git
cd deployment/scripts

./check-prerequisites.sh                # first pass, with defaults
./configure-oapip.sh                    # answers -> generated/values.yaml
./check-prerequisites.sh                # second pass, with your values
./deploy.sh --chart ../../helm-chart
./validation.sh
```

Then list the deployed processes:

```bash
curl https://<your-ogc-api-host>/ogcapi/processes
```

Every prompt, value and check is documented in the
[`deployment` README](https://github.com/cgi-italy-insula-processing/deployment#readme).

## Scope

**Supported.** OGC API - Processes Part 1 v2 (Core) with asynchronous execution and
results by reference, and Part 2 v1 (Deploy, Replace, Undeploy) with CWL Application
Packages.

## Related

| Link | What it is |
|------|------------|
| [Insula Processors](https://github.com/cgi-italy-insula-processors) | Sister organization: turn your own processor repository into a scanned container image and an Application Package deployed to Insula. |
| [EOEPCA+ OAPIP deployment guide](https://deployment-guide.docs.eoepca.org/current/building-blocks/oapip-engine/) | The EOEPCA reference for the processing building block. |
| [OGC API - Processes](https://ogcapi.ogc.org/processes/) | The standard the `ogc-api` service implements. |
| [OGC Best Practice for EO Application Packages](https://docs.ogc.org/bp/20-089r1.html) | How a processor is described in CWL. |

## Feedback and license

Questions and bug reports: open an issue in the relevant repository.

`insula` and `ogc-api` are released under the
[Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0). `minio` redistributes
upstream MinIO under its original Apache License 2.0, with MinIO's `LICENSE` and `NOTICE`
files. The `LICENSE` file in each repository is authoritative.
