# Work In Progress
** Not intended for production env's

## Fun with Config Controller and GitOps

For some fun I decided to put together a little example tutorial on some of the things I've been playing around with, namely Kubernetes [Config Connector](https://cloud.google.com/config-connector/docs/overview) and [Config Controller](https://cloud.google.com/anthos-config-management/docs/concepts/config-controller-overview). Config Connector is a service that lets you define GCP resources as YAML so you can deploy GCP resources from a Kubernetes Cluster. This can be installed in either GKE or other [Kubernetes Distibutions](https://cloud.google.com/config-connector/docs/how-to/install-other-kubernetes). This lets you start managing Google Cloud services like you would other Kubernetes resources which is something I've really been enjoying. You can see the full list of supported resources [here](https://cloud.google.com/config-connector/docs/reference/overview)

Initially the probably I have had with this mean is the default requirement of needing a Kubernetes cluster to act as a management cluster, which you would need to manage and support. I've found this extends to other services the use the Kubernetes Resource Model to manage infrastucture. It's not the end of the world but it's something that can add some overhead and complexity. This is why I'm rather excited about Config Controller which is a new service in Google CLoud which pre-loads a GKE cluster with Config Connector and is managed by Google instead of by me or you! To me this is a great way of solving that chicken and egg situation of needing a cluster to get started. 

The main thing I wanted to test was to see if I could bootstrap a Kubernetes Cluster and wire it up with a GitOps agent to sync with a repository without having to use `kubectl` on it and deploy some applications. The apps I want to deploy for this demo are Falco and Falco Sidekick for a few reasons. First off Falco is a pretty fantastic tool in your Kubernetes Security toolkit (used by folks like Shopify and Gitlab) for doing runtime security and second is Falco Sidekick which is a companion tool for sending Falco alerts to other services in your ecosystem like Pub/Sub! The main reason I wanted to use Falco and Falco Sidekick in action is getting the PubSub integration using [WorkloadIdenity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) for authentication. The nice part to doing this is you won't need to generate a Service Account key and insert that into the Falco Sidekick pod rather but instead using GCP's IAM give the Falco Sidekick K8s SA access to GCP IAM roles.

In order to setup GitOps on the newly generated cluster we will be using the GKE Hub KCC resources ([GKEHubFeature](https://cloud.google.com/config-connector/docs/reference/resource-docs/gkehub/gkehubfeature), [GKEHubFeatureMembership](https://cloud.google.com/config-connector/docs/reference/resource-docs/gkehub/gkehubfeaturemembership), and [GKEHubMembership](https://cloud.google.com/config-connector/docs/reference/resource-docs/gkehub/gkehubmembership)) to configure Config Management. This will sync with the Source Repo that gets created from Config Controller and once you push the demo code to it. 

Finally to help with the distibution of the configs I've been kicking the tires on the [kpt](kpt.dev) tool. 

So if everything goes according to plan you will create a Config Controller instance to deploy a Private GKE cluster which will deploy Falco and Falco Sidekick using Config Management, PubSub (topic and subscription), IAM Roles, Service Accounts and a Source Repo instance to your Google Cloud project.


## Deploying The Infrastructure

### Config Controller

Without further adoo lets create a config controller instance in your GCP project to get started. This should take about 20 minutes or so. This is currently only available in us-central1 and us-east1 regions. I'll be using east1 because that's closest to where I am.

```
gcloud anthos config controller create main --location us-east1
```

Once this is compelete you can get access to the newly created Config Controller cluster by running `gcloud config controller get-credentials main --location us-east1`. Once in the cluster we can start using out `kubectl` commands. Before we do that we'll want to give Config Controller permissions to start creating resources in the project. 

```
export PROJECT_ID=<your_project_id>
gcloud config set project $PROJECT_ID
```

```
  gcloud iam service-accounts create connector
  gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:connector@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/owner"
gcloud iam service-accounts add-iam-policy-binding \
connector@${PROJECT_ID}.iam.gserviceaccount.com \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
    --role="roles/iam.workloadIdentityUser"
```

### Config Connector Deployments

With these permissions the Controller will be able to spin up the necessary GCP resources for the demo. Assuming you ran the prior `gcloud anthos ..` we can move on to the next step of getting our configs ready. These permissions are admitedly wider in scope than they should be in production environments.

In order to get the configs we'll be using `kpt` which come pre-installed in Cloud Shell but if you are running it locally the installation guide can be found [here](https://kpt.dev/installation/).  To get the confings run the following command which will pull the configs into a new directory in you current path `kpt pkg get https://github.com/cartyc/config-controller-example.git/configs`.

Next we'll want to update the setters file and insert your project id in the data section
```
data:
  PROJECT_ID: YOUR_PROJECT_ID
```

Once that is complete run `kpt fn render` to apply the setters file and then run `kpt live apply` to apply the configs.

This will spin up a GKE cluster with Config Sync installed and targetting the `deploy` directory of the Source Repo that is being created along with a Pub/Sub instance Falco Sidekick will be sending the Falco Alerts too (the Falco Sidekick UI is also enabled). 

The services will take a while to spin up and you can keep an eye on them by running `kubectl get gcp --namespace config-controller`. For now we're just concerned with whether or not the Source Repository is only so we can push our configs there. This will take a few minutes as the API gets enabled but you can check on it with `kubectl get sourcereporepository`. Once the service reads ready we can push the configs!

### Falco and Falco Sidekick

While that is being created we can start setting up that repository. To get the configs for it we will pull another `kpt` package `kpt pkg get https://github.com/cartyc/config-controller-example.git/falco-sync` to work on. The configs here are already set up and the only change you'll need to do is update the `kustomization.yaml` to make sure the `falco-falcosidekick` serviceaccounts gets annotated with the information it needs to make sure workloadidentity works right.

```
patch: |-
    - op: add
    path: "/metadata/annotations"
    value:
        iam.gke.io/gcp-service-account: falco-sidekick@${PROJECT_ID}.iam.gserviceaccount.com <-- replace ${PROJECT_ID} with the target projects ID
```

Once you have done that hopefull the Source Repo is created, this takes a few minutes as it needs to enable the API first. If it is you can continue below or go get a refreshment and come back in a few minutes.

```
git init 
git add origin https://source.developers.google.com/p/${PROJECT_ID}/r/falco-sidekick
git push origin --all
```

After a few minutes everything should light up and you'll have Falco up and running with Falco Sidekick sending events to Pub/Sub using WorkloadIdentity for authentication! 

## Wrap-Up

The part about this process is that aside from the work we did in the config controller cluster we didn't have to touch Kubernetes directly and we used WorkloadIdentity to enable our services to access Source Repository and PubSub with needing to create keys or tokens and store them in Git or manually add them to the cluster. That is something I am very excited about! If you already have a repo in github or gitlab you can [mirror](https://cloud.google.com/source-repositories/docs/mirroring-repositories) them into Source Repository and use that as git delivery mechanism.

I would really like to revist this and explore the use of Cloud Deploy to deliver the Infrastructure side of things and fully avoid any `kubectl` work and leave it up to automation.