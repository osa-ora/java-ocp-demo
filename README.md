## Java Apps Deployment Options on OpenShift 

We have plenty of options to deploy Java Application to OpenShift, in this demo we will see some of these deployment options, this repo is fork from my previous Git repo: https://github.com/osa-ora/simple_java_maven

### DEPLOYMENT OPTION 1: Using S2I from the Console

Go to OpenShift Developer Console, Select Java from the catalog and click on Create: 

<img width="335" alt="Screenshot 2024-07-09 at 12 17 58 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/0298e324-e05e-48e2-9077-c8201ae4f486">

Select Java version, fill in the Git Repo location "https://github.com/osa-ora/java-ocp-demo" and application name:

<img width="686" alt="Screenshot 2024-07-09 at 12 18 57 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/c0cd2c0b-2c1a-426e-b668-5d6c061c2f4c">

Note: the repository renamed as java-ocp-demo

Select Build options as "Builds" and click on create.

<img width="678" alt="Screenshot 2024-07-09 at 12 19 44 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/7cade55d-b0ad-4e68-9540-d793ed8331d0">

Note: If you are using private Maven repository, then you can just add an environment variable; MAVEN_MIRROR_URL to point to this private artifact repository.
It can be added as an entry to the .s2i/environment file or as environment variable to the build object.

The application will built and deployed into OpenShift:

<img width="729" alt="Screenshot 2024-07-09 at 12 22 32 PM" src="https://github.com/osa-ora//assets/18471537/af7dddf8-ed90-4b18-9b51-949083dac369">

And you can just test it by using the route:

<img width="391" alt="Screenshot 2024-07-09 at 12 23 05 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/dbc66dbb-9403-4128-b07b-6d04c0fb708f">

You can use that balance URL to get some dummy data from that service.

### DEPLOYMENT OPTION 2: Using Tekton Pipeline from the Console

Follow the same process but during the selection of Build Options select "Pipeline" option as following:

<img width="695" alt="Screenshot 2024-07-09 at 12 24 45 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/35b5efe4-af4d-421f-b1ec-2468ad997eb8">

Follow the progress on the pipeline execution and once successfully finished, check the application deployment status.

<img width="764" alt="Screenshot 2024-07-09 at 12 30 07 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/ef121a35-46c9-4f26-ac39-79ff05d454b2">

You can then enrich the pipeline by adding more sequnetial or parallel tasks as in the folder cicd where you can create secret for a slack channel webhook and enrich the pipeline by test and coverage report as well, you can also add sonar-qube task for static code analysis.

To add slack task, create the secret named my-slack-secret with the webhook token (see the cicd folder) then add the send-slack-webhook task (and configure the secret name and message as followong: 
```
Pipeline execution results $(tasks.status) for $(params.GIT_REPO) -  $(params.GIT_REVISION)
```

<img width="1188" alt="Screenshot 2024-09-12 at 12 23 45 PM" src="https://github.com/user-attachments/assets/f7e301f7-1b40-4f5b-b31d-9760cadd7294">

** Adding SonarQube scanner:

We need to install sonar qube and custom tekton sonar qube scanner job that take the login token first (assuming we are using dev project)

```
//SonarQube
oc process -f https://raw.githubusercontent.com/osa-ora/java-ocp-demo/refs/heads/main/cicd/sonar-qube-template.yaml | oc create -f - -n dev

//Tekton Task: 
oc apply -f https://raw.githubusercontent.com/osa-ora/java-ocp-demo/refs/heads/main/cicd/custom-tekton-sonar.yaml -n dev
```
Create a project in SonarQube and get the project key, token, and url then go to the pipeline and edit it to add parallel task to the test task and select the my-sonar-scanner task 


<img width="1477" alt="Screenshot 2024-09-17 at 10 48 49 AM" src="https://github.com/user-attachments/assets/8028ee81-ce41-4f47-8cdd-fe3a1a610165">



### DEPLOYMENT OPTION 3: Using Binary Build

Execute the following commands from your local machine 

Optionally you can install the dependencies, and in that case it will be a pure binary build, otherwise the builder image will install the dependencies for you.
```
mvn clean package 
```

Deploy the application as a binary build.
```
oc new-project dev
oc new-app --image-stream=openshift/java:11 --name=my-java-app .
mkdir deploy
cp ./target/demo-0.0.1-SNAPSHOT.jar ./deploy/.
cd deploy
oc start-build my-java-app --from-dir=.
oc expose service/my-java-app
```

### DEPLOYMENT OPTION 4: Using Builds for OpenShift:

First you need to install the Builds for OpenShift Operator:

<img width="276" alt="Screenshot 2024-07-08 at 11 54 49 AM" src="https://github.com/osa-ora/angular-demo/assets/18471537/f2f5d78d-e95a-43f6-a9bf-a5fb4a70f6df">

Once, installed create an instance of "Shipwright Build", keep the default

<img width="696" alt="Screenshot 2024-07-08 at 12 32 03 PM" src="https://github.com/osa-ora/angular-demo/assets/18471537/ea1641f3-df9d-4dfd-86bd-9c27ade1e177">

Now, you can deploy the application by creating a Shipwright build either from the console or from the command line:

```
//create an openshift project
oc new-project dev

//create shipwright build for our application in the 'dev' project
//our project code is in the root of the Git repo, otherwise we could have used '--source-context-dir="docker-build"' flag to specify the context folder of our application.
shp build create java-build --strategy-name="source-to-image" --source-url="https://github.com/osa-ora/java-ocp-demo" --output-image="image-registry.openshift-image-registry.svc:5000/dev/java-app" --builder-image="image-registry.openshift-image-registry.svc:5000/openshift/java:11"

//start the build and follow the output
shp build run java-build --follow

//create an application from the container image
oc new-app java-app

//expose our application
oc expose service/java-app

//test our application is deployed ..
curl $(oc get route java-app -o jsonpath='{.spec.host}')/

```

You can do the same from the Dev Console

Go to "Builds" section and click on Create and select "Shipwright Build"

<img width="1189" alt="Screenshot 2024-07-08 at 12 38 53 PM" src="https://github.com/osa-ora/angular-demo/assets/18471537/9eb76e9b-1dc0-41a2-992e-ea49985ba513">

Post the following content:

```
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  name: java-build
  namespace: dev
spec:
  output:
    image: 'image-registry.openshift-image-registry.svc:5000/dev/java-app'
  paramValues:
    - name: builder-image
      value: 'image-registry.openshift-image-registry.svc:5000/openshift/java:11'
  source:
    git:
      url: 'https://github.com/osa-ora/java-ocp-demo'
    type: Git
  strategy:
    kind: ClusterBuildStrategy
    name: source-to-image
```

Click on Create and then click on "Start Build", follow the logs:

<img width="813" alt="Screenshot 2024-07-09 at 12 44 10 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/1bfe4d5f-5c4d-4ae0-bf6e-8f4842f88236">

Now, you can deploy the appliation using "oc new-app java-app" or from the console using container image option:

<img width="681" alt="Screenshot 2024-07-09 at 12 45 00 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/b111f1e7-6d4b-411a-8387-17a232a06a25">

Test the application route and we are done!

If you are building this using private repository artifact, just add to the file ".s2i/environment" the following entry:
```
MAVEN_MIRROR_URL={the private repository artifact URL}
```

<img width="626" alt="Screenshot 2024-07-09 at 12 46 14 PM" src="https://github.com/osa-ora/java-demo/assets/18471537/e6f8f223-d813-41b8-aed6-75aa2e0194b1">

### DEPLOYMENT OPTION 5: Using the Container Imaage

If we managed to import the container image inside OpenShift or if we have it in any image registry accessible from OpenShift, then we can go to the Add section and select container image and select if this will be internal image registry or external registry .. and let OpenShift do the magic of creating all the necessary files.

Let's try this with our current dev/java-demo image that is created in any of the previous deployment steps.

<img width="696" alt="Screenshot 2024-09-19 at 3 32 22 PM" src="https://github.com/user-attachments/assets/59d62aca-f9e3-43d1-80d2-c298bd71e5ae">

We refer to the internal registry with "image-registry.openshift-image-registry.svc:5000/{openshift-project-name}/{image-name}:{tag optionally}

<img width="694" alt="Screenshot 2024-09-19 at 3 33 41 PM" src="https://github.com/user-attachments/assets/1a9326b0-7dca-472b-a50f-33a02c43772d">

Click on create to create this new application.

<img width="754" alt="Screenshot 2024-09-19 at 3 34 54 PM" src="https://github.com/user-attachments/assets/7f9ce3e8-3776-48ef-9e92-ffb996261c7d">

Note: it will be deployed quickly as no build is required.

---
### DEPLOYMENT OPTION 6: Deployment using GitOps Approach

We need to capture our configurations files and store them in a Git repository, for that we will capture the deployed app yaml files and store them in this repo in a folder called gitops.

The easy way to create these files is to follow the following:
1) Create App Source of Truth:
- Open Deployment Config: Remove status section, remove metadata except name and label, and ave the file as "deployment.yaml"
- Open Service Config: Remove status section, remove metadata except name and label, remove clusterip, and save the file as "service.yaml"
- Open Route: Remove status section, remove metadata except name and label, remove host, and save the file as "route.yaml"
- If the application is using configMap or secret we can capture them and remove the status, metadata except name and label and save them with their corresponding names or group configs or secrets.

2) Deploy OpenShift GitOps Operator:
- Go to the OperatorHub and install the operator.
- It will install a default argocd instance, get the route url (from argocd-server) and admin user password (from argocd-cluster secret).
- Login to it using admin/password
- Initially there will be no application configured.
  
  <img width="1117" alt="Screenshot 2024-09-19 at 1 27 13 PM" src="https://github.com/user-attachments/assets/179f525d-3344-43d0-b4ee-4818be066267">

  
3) Configure our App for GitOps:
You can configure the application for deployment from the GUI or using a yaml file directly to OCP (or in the operator section)

Define App Configs and sync prooperties:

<img width="1171" alt="Screenshot 2024-09-19 at 1 30 12 PM" src="https://github.com/user-attachments/assets/e01ba82e-fb40-473c-b43e-6c9ceaf7b78a">

Define GitOps source files:

<img width="1153" alt="Screenshot 2024-09-19 at 1 33 14 PM" src="https://github.com/user-attachments/assets/71e3f5a9-bbf6-48b6-838d-1ae28060e060">

Note: the repository renamed as java-ocp-demo

Define the destination taerget:

<img width="1155" alt="Screenshot 2024-09-19 at 1 31 26 PM" src="https://github.com/user-attachments/assets/382375dd-9d32-4136-8b2c-7d6ccbd2141f">
 
Note: this destination means the local cluster of argocd.

Click on Create....

After a few seconds, you can see the application is healthy and synced.

<img width="836" alt="Screenshot 2024-09-19 at 1 33 47 PM" src="https://github.com/user-attachments/assets/4eb2872c-dd5e-4746-899c-535bf88bb04d">

Note: the repository renamed as java-ocp-demo

Capture the application yaml file from either the Argocd GUI or its Operator.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: maven-app-gitops
spec:
  destination:
    namespace: dev
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    path: gitops
    repoURL: 'https://github.com/osa-ora/java-ocp-demo'
    targetRevision: main
  syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Store it to the GitRep (we stored it in the cicd section as gitops-app.yaml file)
Test the application by clicking on the route to make sure our application is working fine.

4) Modify the Tekton Pipeline: (that we used in Deployment option 2)

Let's now modify the Tekton Pipeline to remove the App deployment part and replace it with image tag and gitops deployment.

- Add parameter to the pipeline called "APP_VERSION" with default as 1.0
- Modify the deploy step name to "tag-image" and replace the command with
```
  oc tag $(params.APP_NAME):latest $(params.APP_NAME):$(params.APP_VERSION)
```
  so now everytime we run the pipeline we need to specify a version number to have a new image tag.
- Save the pipeline.
- Run the pipeline with APP_VERSION 1.0 then 1.1 then go to ImageSteam and see the different tags that are there ...
  
  <img width="877" alt="Screenshot 2024-09-19 at 2 14 27 PM" src="https://github.com/user-attachments/assets/862fd469-00db-4a6d-b89a-7a55eb3cfe6b">

- Now go to deployment.yaml file in our gitops folder and update it with any of these tags.
- Check the progress on the argocd and check the deployed "deployment" object in openshift and check the reference image before and after that change.
- Now with every new build we can increment the version or keep it the same and update the gitops files if needed.

  <img width="384" alt="Screenshot 2024-09-19 at 2 17 21 PM" src="https://github.com/user-attachments/assets/22c56e0b-8937-430d-b5ef-fba918bbbbdd">

  Now, you can see the full pipeline built in this demo in the cicd folder: full_tekton.yaml

<img width="1195" alt="Screenshot 2024-09-19 at 3 01 58 PM" src="https://github.com/user-attachments/assets/e42fb0fc-28ce-423a-a13c-4c4c55520168">

We send initial slack notification that the build started for a specific revision/branch/tag then fetch the code, build, test, run sonar qube (if flag is set to true), tag the image, and finally send a slack notification with the results.
Once a new version is deployed, you can just edit the gitops files so the argocd can reflect it on OpenShift.
You can also modify anything you need to change, like number of replica and it will be reflected, if you need to use auto-scaling, then delete the replica count from the deployment.yaml file and let OpenShift manage it based on the configured auto-scaling capabilities.

---
### DEPLOYMENT OPTION 7: Deployment using Helm

From OpenShift Web Terminal, write the following command:

```
oc new-project test2 //or any other project name
helm install my-java-app https://raw.githubusercontent.com/osa-ora/java-ocp-demo/main/helm-chart/releases/java-ocp-demo-0.1.0.tgz
```
<img width="1491" alt="Screenshot 2025-03-16 at 6 19 40 PM" src="https://github.com/user-attachments/assets/6f7889d4-a8ae-4bcc-a81a-e026d5d5f5f7" />

The application components will be deployed successfully in this project.

<img width="1194" alt="Screenshot 2025-03-16 at 6 21 12 PM" src="https://github.com/user-attachments/assets/5d44cc49-f5eb-482a-966f-caf06449953f" />

You can edit the applicaton by go to Helm --> Upgrade .. 

<img width="1187" alt="Screenshot 2025-03-16 at 6 21 58 PM" src="https://github.com/user-attachments/assets/9f220997-a30a-47a6-b9c9-4a605a20eaf1" />

You can edit the values that specified during the release build for example change the replica count into 2, this will create a new revision.

<img width="1183" alt="Screenshot 2025-03-16 at 6 24 16 PM" src="https://github.com/user-attachments/assets/aa2e7f40-04ea-454e-9739-b6e6d726ccca" />

Now if you go the deployment you can see 2 replica count. 

<img width="697" alt="Screenshot 2025-03-16 at 6 24 45 PM" src="https://github.com/user-attachments/assets/b488de5b-a1b0-4b91-83e8-4f6d3db46456" />

You can rollback to the previous release by selecting any previous revisions that you have created by selecting rollback:

<img width="1170" alt="Screenshot 2025-03-16 at 6 25 47 PM" src="https://github.com/user-attachments/assets/36647582-5f04-4e70-b1f8-738a179b07eb" />

Select an old revision e.g. revision 1, you'll notice the replica count is back to 1.

Note: To get the release revision file that we have deployed, you need just to clone the repository, execute "helm package . " while you are inside the "helm-chart" folder, then upload the helm release file into the release folder "java-ocp-demo-0.1.0.tgz", you can version your helm by editing "Chart.yaml" and change the current version "version: 0.1.0" if you have changed different yaml file contents or values.

Note: the helm chart are using Quay.io hosted image in the location: quay.io/ooransa/java-ocp-demo:latest but you can refer to any image registry visible and accessible by OpenShift cluster.

This image was uploaded from the s2i using the skopeo command. 

```
skopeo copy --all \
  --src-creds "$(oc whoami):$(oc whoami -t)" \
  docker://default-route-openshift-image-registry.apps.cluster-........opentlc.com/test/java-ocp-demo:latest \
  docker://quay.io/ooransa/java-ocp-demo:latest \
  --dest-creds "ooransa:my_token"
```
Note that in order to expose OpenShift image registery you'll need to patch it:

```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```
See: https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/registry/securing-exposing-registry#registry-exposing-default-registry-manually_securing-exposing-registry

This covers various options to deploy Java application into OpenShift.

