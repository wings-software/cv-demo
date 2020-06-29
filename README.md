# CV Demo Setup

## Create a kubernetes cluster
- One node is enough, with minimum **2 CPU** and **6 GB** memory
- You will need more capacity if you plan to put a delegate in the cluster

## Add kubernetes cloud provider in Harness ([docs](https://docs.harness.io/article/whwnovprrb-cloud-providers#kubernetes_cluster))
- In Harness, add kubernetes cloud provider for the cluster, called `cv-demo`
- This could use service account credentials and master endpoint if you want to use a delegate from outside the cluster, or
- Otherwise, install a delegate into the cluster first, then create the cloud provider

## Add Dockerhub artifact repo connector ([docs](https://docs.harness.io/article/tdj2ghkqb0-add-docker-registry-artifact-servers))
- Add Dockerhub artifact repo connector called `public-docker`
- Use URL `https://index.docker.io/v2/`
- The images are public but docker credentials are needed for the connector
  - Note: if you don’t name the connector the same the artifact source won’t sync.<br>
    If that happens you can add the artifact source yourself.<br>
    cv-demo service: `harness/cv-demo`<br>
    cv-demo-ui service: `harness/cv-demo-ui`

## Clone app from git ([docs](https://docs.harness.io/article/ay9hlwbgwa-add-source-repo-providers))
- Create a new app called `cv-demo`
- Setup bi-directional git sync while the app is still empty
  - Add git connector and create webhook for bidirectional
  - In git, add the webhook and change content type to `application/json`
- Clone our **cv-demo** repo:<br>
  `git clone https://github.com/wings-software/cv-demo.git`
- Copy cv-demo configuration into your own sync repo:<br>
  `cp -R cv-demo/Setup/Applications/cv-demo/* my-cv-demo/Setup/Applications/cv-demo/`
- `cd my-cv-demo`
- Remove Service Guard config for now (we’ll add it back later):<br>
  `rm -rf Setup/Applications/cv-demo/Environments/cv-demo/Service\ Verification/`
- `git add -A`
- `git commit -a`
- `git push`
- Check that your app was synced in the Harness UI
  - Check for errors in *Configuration as Code*

## Reserve static IPs
- Reserve static IPs (see instructions for [AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html#using-instance-addressing-eips-allocating), [GCP](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address#reserve_new_static), or [Azure](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-public-ip-address#create-a-public-ip-address)) for:
  - Ingress controller
  - Prometheus
  - Elastic Search
- Optionally reserve DNS name for the ingress controller static IP

## Update Harness environment `cv-demo`
- Edit values YAML override for `nginx-ingress-controller` service and override:<br>
	`loadBalancerIP: <ingress-static-ip>`
- Edit values YAML override for cv-demo service and override:<br>
```
  env:
    config:
      ALLOWED_ORIGINS: http://<ingress-static-ip or dns>
```
- Edit service variable override for **cv-demo-ui** service and override:<br>
	`baseUrl: http://<ingress-static-ip or dns>`
- Edit values YAML override for **prometheus** service and override:<br>
	`loadBalancerIP: <prometheus-static-ip>`
- Edit values YAML override for **elastic-search** service and override:<br>
	`loadBalancerIP: <elk-static-ip>`

## Execute Setup pipeline
- Execute the pipeline **CV Demo - Cluster Setup**
- Select `stable` artifact for **cv-demo** service and `latest` for **cv-demo-ui**
- This deploys the Nginx ingress controller, prometheus, and elastic search, as well as a baseline cv-demo backend and the cv-demo-ui for controlling it

## Create verification connectors ([docs](https://docs.harness.io/article/r6ut6tldy0-verification-providers))
- Create `ELK-cv-demo` with URL `http://<elastic-search-ip>`
- Create `Prometheus-cv-demo` with URL `http://<prometheus-ip>:8080/`

## Edit `cv-demo-canary` workflow
- In Prometheus step, select your Prometheus connector
- In ELK step, select your ELK connector

## Execute workflow
- Execute workflow `cv-demo-canary`
- Select `verify_canary = yes`
- You can either select the unstable image tag, or:
  - In browser, visit `http://<ingress-static-ip or dns>`
  - Adjust the canary log and metric error rates and values via the UI
- *Observe Verification steps detecting the anomalies*

## Setup 24/7 Service Guard 
[24/7 docs](https://docs.harness.io/article/dajt54pyxd-24-7-service-guard-overview)<br>
[prometheus docs](https://docs.harness.io/article/i9d01kf32g-2-24-7-service-guard-for-prometheus)<br>
[Elasticsearch docs](https://docs.harness.io/article/564doloeuq-2-24-7-service-guard-for-elasticsearch)

- Copy Service Guard configuration into your own sync repo:<br>
  `cp -R cv-demo/Setup/Applications/cv-demo/Environments/cv-demo/Service\ Verification my-cv-demo/Setup/Applications/cv-demo/Environments/cv-demo`
- `git add -A`
- `git commit -a`
- `git push`
- Check that 24/7 Service Guard is now configured in Continuous Verification
  - Check for errors in *Configuration as Code*
- Open the ELK configuration
  - Select baseline as last 30 minutes
  - Submit
- In browser, visit `http://<ingress-static-ip or dns>`
- Adjust the primary log and metric error rates and values via the UI
- *Wait some time and observe 24/7 Service Guard detecting anomalies*


