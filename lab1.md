# Step 1. Installing the Agent
## 1. To get started we need to set a few permissions. The first one is to create a role scoped to the cluster. This role will get and update the ConfigMaps, list and watch the Events, get, update, and create Endpoints, and list the componentstatuses resource. Run the following command on the master:
```
kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrole.yaml"
```
## You can discover what this role is giving access by opening the url above in your browser. But defining the role alone doesn’t really do anything yet
### 1. Next create the service account:
```
kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/serviceaccount.yaml"
```
### 2. The previous two steps are then linked together in the clusterrolebinding which binds the service account to the clusterrole:
```
kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrolebinding.yaml"
```
### 3. The next step is to create the manifest for the Datadog agent and apply it to the cluster. You can find the manifest in the documentation, but one is already in the k8s-yaml-files directory, located in the IDE tab on the left. Click on the file to see what’s in there.
<br>

### If you compare it with the documentation you may notice that the first block that defines the api-key as a secret is not included. This step has already been completed as part of the provisioning of the training environment.
<br>

### 4. Apply this manifest using the following command:
```
kubectl apply -f k8s-yaml-files/datadog-agent.yaml
```
### 5. kubectl get daemonset
```
kubectl get daemonset
```
### You should see a list of how many agents are installed and running.

### 6. Open the datadog-agent.yaml file. Notice that many of the configuration options for the agent are defined with environment variables.
<br>

## Step 2. Adding an Integration

### 1. When this lab began, you were provisioned credentials for a Datadog account. Run the `creds` command in the terminal to retrive those credentials, then log into the Datadog account and take a look around.

### 2. Make sure you take a look at the Event Stream.

### Here you will see an event for each of the containers as they appear in the Datadog interface. Note that it could take a few minutes for them to show up in the interface for the first time.

### 3. Navigate to the Integrations List.

### 4. Enable the tile for the integration for Postgres.

### Normally you would follow the instructions displayed in the Configuration tab in the tile. Most of those steps have already been completed and all that is necessary here is to enable the tile.

### 5. Now go to the Dashboards list and find the Postgres - Overview dashboard.

### 6. Even if you wait a long time, no metrics will appear here. This is because the Agent is not reporting any Postgres metrics.

### 7. In the lab environment IDE tab to the left, open the `postgres-deploy.yaml` file. Scroll down to line 19.

### 8. There is a section for annotations; uncomment each of these lines. Then save the file by clicking the disk icon in the IDE tab.

### When the hashes are removed, annotations: should be at the same indent level as labels: and each of the three annotation lines will be indented two spaces from annotations.

### Annotations are how you configure the Datadog Agent to work with one of the integrations. Here we are telling the Agent to use the Postgres check, with the corresponding host, port, username, and password.

### Since we can't possibly know what the host and port are going to be when we write the yaml file, the %%HOST%% is a placeholder that is replaced automatically at run time.
<br>

### 9. Now apply the Postgres deployment again:
```
kubectl apply -f k8s-yaml-files/postgres-deploy.yaml
```

### 10. If you take a look at the Datadog dashboard now, even if you wait a few minutes, you still won't see anything. We have configured the Postgres Integration, but it's not working.

### 11. Just to make sure all the pods are running, lets look at the results of:
```
kubectl get pods
```
### Looks like everything is running.

### 12. Move on to the next section to find a few methods for seeing what's wrong.

## Step 3. Troubleshooting

## In this exercise, just like in the real world, problems can come up. Unless you know where to look, things can be frustrating. There are a few different commands you can use to try to troubleshoot your Kubernetes environment.

### 1. The most common problem users have when working with Datadog is getting the API key wrong. One way to verify this is to open the Organization Settings > API Keys section of the Datadog application.

### 2. Now open the datadog-agent.yaml file in the editor. The Datadog agent gets the API key from an environment variable. In the case of this YAML file, the environment variable is being populated from a kubernetes secret. To get the secret you can run:
```
kubectl get secret/datadog-api -o json
```
### Here is a command that will filter down the output and base64 decode the api key all in one:
```
kubectl get secret/datadog-api -o json | jq .data.token | base64 -d -i
```
### 3. Of course, we know that the API key isn't the problem because we are getting some data into Datadog, just not the Postgres metrics.

### 4. The next thing to look at is the status of the Datadog Agent.

### 5. We need to run the command `agent status` on the Datadog Agent pod. To get the name of the pod, you can run:
```
kubectl get pods --no-headers -o custom-columns=":metadata.name" | grep datadog
```
### 6. Once we have that, `run kubectl exec <pod name> -- agent status`. Or just put it all together and run:
```
kubectl exec `$(kubectl get pods --no-headers -o custom-columns=":metadata.name" | grep datadog | head -n 1) -- agent status`
```
### 7. Now scroll up to the Running Checks section that mentions postgres.

### 8. You can see there is an error: password authentication failed. Take a look at the yaml file for postgres-deploy.

### 9. The problem is that the password was set to `datadog` and should in this case be `postgres`. Make the change to the password in the `ad.datadoghq.com/postgres.instances` annotation of `postgres-deploy.yaml` and redeploy postgres:
```
kubectl apply -f k8s-yaml-files/postgres-deploy.yaml
```
### In this exercise we chose to set the password in the yaml file to make it easier to update. But in the real world you should consider storing passwords in Kubernetes' secrets. For instance, the API key that you accessed above was stored using this command: `kubectl create secret generic datadog-api --from-literal=token=<APIKEYHERE>`

### 10. Now re-run the Agent status command above. It may take a few minutes for the postgres check to show up again.

### Note that the deployment needs to stop and restart, and then the agent needs to update the check so even if everything seems to work but you are still getting an error, give it a little longer and try again.

### 11. When everything looks normal, take a look at the dashboards in the Datadog application and you'll find information being reported.