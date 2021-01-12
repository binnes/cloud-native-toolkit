# Running the Day One instructions

This page outlines the additional steps needed to run through the [day 1 instructions](https://cloudnativetoolkit.dev/getting-started-day-1/deploy-app)

## Setup additional steps - before running the oc sync command

Currently the setup doesn't fully create all the needed kubernetes objects.

You need to update and then apply the following configuration files in the kubernetes folder of this repository.  The only updates needed are to fix up the IP address.  This should be the IP address returned by the ```minikube ip``` command :

- ibmcloud-config.yaml
- registry-config.yaml

To apply the files simply user the kubectl or oc command:

```shell
kubectl apply -f ibmcloud-config.yaml
kubectl apply -f registry-config.yaml
```

## Fixing up the Tekton Pipeline configuration

The command line tool doesn't create all the required tekton configuration or the Gitea webhook.  In addition some of the created pipeline configuration assumes OpenShift running on the IBM Cloud, so need to be updated to run locally.

1. Apply the following 3 files.   You need to update the content to deploy the configuraion to your project namespace (bi-dev in the files in the repo kubernetes folder)

    ```shell
    kubectl apply -f tekton-gitea-clusterBinding.yaml
    kubectl apply -f test-event-listener.yaml
    kubectl apply -f test-eventListener-ingress.yaml
    ```
2. The generated TriggerTemplate for the project is not correct.  the params path to retrieve the git configuration values needs to be tt.params rather than just params.  To fix this use the Kubernetes dashboard app or command line to update the Custom Resource Definition for the triggertemplates object created for the project.  An additional config property also needs to be added to allow the ingress type to be overridden.  By default an OpenShift route will be created rather than a kubernetes route.  The params section should be as seen below:

    ```yaml
    spec:
      params:
        - name: git-url
          value: $(tt.params.gitrepositoryurl)
        - name: git-revision
          value: $(tt.params.gitrevision)
        - name: scan-image
          value: 'false'
        - name: deploy-ingress-type
          value: ingress
    ```

3. Again in the TriggerTemplate object for the project the ServiceAccount *pipeline* should be used to run tasks, as this serivce account has the necessary roles assigned.  Using the default account causes RBAC access failures.  To add the service account modify the TriggerTemplate object for the project and set the serviceAccountName property:

    ```yaml
    resourcetemplates:
      - apiVersion: tekton.dev/v1beta1
        kind: PipelineRun
        metadata:
          generateName: stockbffnode-
        spec:
          params:
            - name: git-url
              value: $(tt.params.gitrepositoryurl)
            - name: git-revision
              value: $(tt.params.gitrevision)
            - name: scan-image
              value: 'false'
            - name: deploy-ingress-type
              value: ingress
          pipelineRef:
            name: stockbffnode
          serviceAccountName: pipeline
    ```

4. The role definition for the pipeline needs to be adjusted as the ingresses resource belongs to the *networking.k8s.io* API Group, not extensions.  The role definition within your project namespace should be:

    ```yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: pipeline
      namespace: bi-dev
    rules:
      - verbs:
          - '*'
        apiGroups:
          - ''
        resources:
          - services
          - pods
          - pods/exec
          - pods/log
          - secrets
          - configmaps
      - verbs:
          - '*'
        apiGroups:
          - apps
        resources:
          - deployments
      - verbs:
          - '*'
        apiGroups:
          - networking.k8s.io
        resources:
          - ingresses
    ```

5. The Tekton Pipeline definition needs to be updated to pass the deploy-ingress-type parameter to the tasks.  Update the Custom Resource Definition for the Pipeline object created for the project.  The project pipeline object should be changed to include :

    ```yaml
    spec:
      params:
        - description: The url for the git repository
          name: git-url
          type: string
        - default: master
          description: 'The git revision (branch, tag, or sha) that should be built'
          name: git-revision
          type: string
        - default: 'true'
          description: Flag indicating that an image scan should be performed
          name: scan-image
          type: string
        - default: ingress
          description: Create an OpenShift route or Kubernetes Ingress configuration
          name: deploy-ingress-type
          type: string
      tasks:
        - name: setup
          params:
            - name: git-url
              value: $(params.git-url)
            - name: git-revision
              value: $(params.git-revision)
            - name: scan-image
              value: $(params.scan-image)
            - name: deploy-ingress-type
              value: $(params.deploy-ingress-type)
          taskRef:
            kind: Task
            name: ibm-setup
    ```
    