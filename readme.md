## How to convert KubeTruth K8s Operator to Helm-based Operator using [Operator SDK framework](https://sdk.operatorframework.io/docs/building-operators/helm/)

[KubeTruth source code](https://github.com/cloudtruth/kubetruth)

### We want to convert the KubeTruth Operator written in Ruby and Helm Chart to Kubernetes Operator that will work on Openshift 4.12 and with as little changes to the source code as possible.

### The first thing we want to do is to make sure KubeTruth works on Openshift.

Follow the direction from [here](https://docs.cloudtruth.com/integrations/kubernetes) to install KubeTruth on Openshift 4.12.

After installing KubeTruth we see the pod is not running with Error status. 

The Pod's log can offer for more info. 
   
````
$ oc logs kubetruth-install-b8867597b-6cdqq
Starting app
/usr/local/lib/ruby/3.0.0/bundler/shared_helpers.rb:105:in `rescue in filesystem_access': There was an error while trying to write to `/srv/app/Gemfile.lock`. It is likely that you need to grant write permissions for that path.
```` 
We see that there's a problem writing to /srv/app/Gemfile.lock

This is a correct behavior in Openshift.

Openshift uses security context constraints [(SCCs)](https://docs.openshift.com/container-platform/4.12/authentication/managing-security-context-constraints.html#security-context-constraints-about_configuring-internal-oauth) to control permissions for the pods in your cluster.

With the exception of default, kube-system, and openshift-operators, namespaces, Openshift put restricted-v2 SCC on pods.

````
$ oc get pod kubetruth-install-777d7d8745-xnxhd -oyaml | grep scc
    openshift.io/scc: restricted-v2
````

So now let's examine restricted-v2 :

````
oc describe scc restricted-v2
````
The reason why we can't write to /srv/app/Gemfile.lock is because the pod runs with userid => 1000740000

````
$ oc get pod kubetruth-install-777d7d8745-xnxhd -oyaml | grep runAsUser
      runAsUser: 1000740000
````

To fix this without making changes to the container's permission, we cadd scc policy for the serviceaccount.

1. Find out pod's ServiceAccountName

````
$ oc get pod kubetruth-install-b8867597b-6cdqq -oyaml | grep serviceAccountName
  serviceAccountName: kubetruth-install
````

2. Use this command to add anyuid to scc 

````
oc adm policy add-scc-to-user anyuid -z kubetruth-install
````

3. Delete the offending pod and you should see a new pod spins up automatically.

4. Examine the pod status. It should be Running.
5. Examine the scc.
````
oc get pod kubetruth-install-777d7d8745-l4d7q -oyaml | grep scc
    openshift.io/scc: anyuid
````
6. Examine runAsUser.
   
   Should return nothing.
   
   ````
   oc get pod kubetruth-install-777d7d8745-l4d7q -oyaml | grep runAsUser
   ````
   This means that there's no restriction on who can run the pod.

### We are ready to build Helm-based Operator with the Helm Chart provided by KubeTruth.

#### A few things to consider

1. The Operator already have a custom Kind associated with the Ruby Operator, so we will create a new Kind with a different name.
   
#### We want to extract any rbac from the helm charts because OLM handles all rbac.

2. Extract the following from the helm charts and move them into the config/rback directory
   * templates/clusterrole.yaml
   * templates/clusterrolebinding.yaml
   * templates/rolebinding.yaml
   * templates/serviceaccount.yaml

3. Extract the following from the helm charts and move it into the bundle/manifests directory
   * crds/projectmapping.yaml
   
4. We need to add SCC anyuid to the clusterrole which is bind to the serviceaccount.

5. The OLM is creating the serviceAccount so remove [this section](https://github.com/cloudtruth/kubetruth/blob/main/helm/kubetruth/templates/_helpers.tpl#L53-L62) of the helpers. 

### Let's begin by creatinig a Helm-based operator.

1. Initialize a Helm Project
````
operator-sdk init --domain cloudtruth.com --plugins helm
````
2. Create the API.

   Group : apps

   Version : v1alpha1

   Kind : KubeTruth

   Fetch the Helm chart from my local disk
````
operator-sdk create api \
 --group=apps --version=v1alpha1 \
 --kind=KubeTruth \
 --helm-chart=/Users/rosecrisp/codebase/kubetruth/helm/kubetruth
````
1. Remove rbac contents from helm-charts/kubetruth/values.yaml [see result](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/values.yaml). This will be handled by OLM.
2. Remove serviceAccount contents from helm-charts/kubetruth/values.yaml. This will be handled by OLM.
3. Extract clusterrole.yaml, clusterrolebinding.yaml, role.yaml, rolebinding.yaml, and serviceaccount.yaml from helm-charts/kubetruth/templates dir [see result](https://github.com/rocrisp/kubetruth/tree/main/helm-charts/kubetruth/templates)
4. OLM handles the rbac, so create serviceaccount.yaml, clusterrole.yaml, and clusterrolebinding.yaml in the config/rbac/ subdir [see here](https://github.com/rocrisp/kubetruth/tree/main/config/rbac)

   Notice the clusterrole with the SCC with anyuid allows anyuid to access the container in the pod [see here](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/kubetruth_install_clusterrole.yaml#L41)
5. Move projectmappings.yaml from helm-charts/kubetruth/crds subdir to config/manifests subdir
6. OLM automatically generate the serviceAccount so the deployment.yaml need to reference the new serviceAccountName by replacing the serviceAccountName in the helm-charts/kubetruth/templates/deployment.yaml to "kubetruth-operator-kubetruth-install" [see here](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/templates/deployment.yaml#L27)
7. Now we want to integrate with OLM to make delivering software very easy. 

   1. We have an extra serviceaccount.
   
   Add the extra serviceaccount to OLM by adding an extra flag --extra-servive-accounts to the Makefile [see doc](https://sdk.operatorframework.io/docs/advanced-topics/multi-sa/).
   The result look like [this](https://github.com/rocrisp/kubetruth/blob/main/Makefile#L157)

   2. Generate files in bundle format Example, search for ["make bundle"](https://sdk.operatorframework.io/docs/olm-integration/generation/)
   
   ````
   make bundle
   ````
   See the directory created by the command [here](https://github.com/rocrisp/kubetruth/tree/main/bundle)

   The manifests directory holds all the generated files from [config/](https://github.com/rocrisp/kubetruth/tree/main/config)
8. Move projectmapping.yaml from helm-chart/kubetruth/crd/ directory to bundle/manifests subdir so OLM can install it automatically. [See here](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/projectmapping.yaml)
9.  The rbac permission from [here](https://github.com/cloudtruth/kubetruth/blob/981d3719a4e1ab6c70e9f8e6c41ed21da06d3acb/helm/kubetruth/values.yaml#L26) is added to csv [here](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/kubetruth-operator.clusterserviceversion.yaml#L95) and [here](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/kubetruth-operator.clusterserviceversion.yaml#L338)
10. There are a few environment variables the Makefile depends. I added a setenv.sh file to make it easy for when you need to build, push containers. [see here](https://github.com/rocrisp/kubetruth/blob/main/setenv.sh)
11. You can execute the setenv.sh with this command,
````
source setenv.sh
````
1.  And now we can build the operator, operator bundle, and deploy it using olm integration with operator-sdk
````
make docker-build docker-push
make bundle-build bundle-push
````
1.  Having the extra serviceAccount means that you're limited to a namespace. [see here](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/kubetruth-operator-kubetruth-install-clusterrolebinding_rbac.authorization.k8s.io_v1_clusterrolebinding.yaml#L13)
    so we create the namespace first,
    ````
    oc new-project kubetruth-operator-system
    ```
2.  Now we can use OLM to deploy our operator.
```
operator-sdk run bundle quay.io/rocrisp/kubetruth-operator-bundle:v0.1.0
```
1.  As the last step, we deploy the operand.
````
oc apply -f config/samples/apps_v1alpha1_kubetruth.yaml
````

Some tools to have when you're trying to test your operator.

This command will remove resources created by OLM, but will not remove clusterrole, clusterrolebinding, and serviceAccount.
````
operator-sdk cleanup kubetruth-operator
````

There are times you need to modify CSV file. This command validas the syntax to make sure all the Ts and Ds are checked.
````
operator-sdk bundle validate ./bundle
````