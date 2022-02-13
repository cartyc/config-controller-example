# Deploying GKE Cluster with Falco Sidekick using Config Controller

First off create a config controller instance in your GCP project, this should take about 20 minutes or so

```
gcloud anthos config controller create main --location us-east1
```

Once this is compelete you can get access to the newly created Config Controller cluster by running `gcloud config controller get-credentials main --location us-east1`. Once in the cluster we can start using out `kubectl` commands. Before we do that we'll want to give Config Controller permissions to start creating resources in the project. 

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

With these permissions the Controller will be able to spin up the necessary GCP resources for the demo. Assuming you ran the prior `gcloud anthos ..` we can move on to the next step of getting our configs ready.

In order to get the configs we'll be using `kpt` which come pre-installed in Cloud Shell but if you are running it locally the installation guide can be found [here](https://kpt.dev/installation/).  To get the confings run the following command which will pull the configs into a new directory in you current path`kpt pkg get <url-to-this-repo>`.

Next we'll want to update the setters file and insert your project id in the data section
```
data:
  PROJECT_ID: YOUR_PROJECT_ID
```

Once that is complete run `kpt fn render` to apply the setters file and then run `kpt live apply` to apply the configs.

This will spin up a GKE cluster with Config Sync installed and targetting the `falco-sync` directory of the Source Repo that is being created along with a Pub/Sub instance Falco Sidekick will be sending the Falco Alerts too (the Falco Sidekick UI is also enabled). 

The services will take a while to spin up and you can keep an eye on them by running `kubectl get gcp --namespace config-controller`. For now we're just concerned with whether or not the Source Repository is only so we can push our configs there. This will take a few minutes as the API gets enabled but you can check on it with `kubectl get sourcereporepository`. Once the service reads ready we can push the configs!

```
git init 
git add origin https://source.developers.google.com/p/${PROJECT_ID}/r/falco-sidekick
git push origin --all
```

After a few minutes everything should light up and you'll have Falco up and running with Falco Sidekick sending events to Pub/Sub for you take action on or use to add more visibility.