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

To fix this, without having to change the container's permission, we can add scc policy for the serviceaccount.

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

1. The Operator already has a CRD so we will create a new Kind with a different name.
   
2. We need to move the rbac from template/ to the config/rback directory
   
3. We need to add SCC Pod permission using anyuid.

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
1. Remove rbac contents from helm-charts/kubetruth/values.yaml [see result](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/values.yaml)
2. Remove clusterrole.yaml, clusterrolebinding.yaml, role.yaml, rolebinding.yaml, and serviceaccount.yaml [see result](https://github.com/rocrisp/kubetruth/tree/main/helm-charts/kubetruth/templates)
3. To get around SCC, we create a serviceaccount, clusterrole, and clusterrolebinding. With this, the pod will be created with anyuid which allows any id to access the container in the Pod. [see here](https://github.com/rocrisp/kubetruth/tree/main/config/rbac)

   The clusterrole has the scc changes needed for the pod to be able to create a container with anyuid allows any id to access the container in the pod [see here](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/kubetruth_install_clusterrole.yaml#L41)

4. The deployment will use the newly created serviceAccount so the container in the pod has the changes for the scc. To make this happen, in the templates/deployment.yaml file modify the serviceAccountName to "kubetruth-operator-kubetruth-install [see here](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/templates/deployment.yaml#L27)
5. Now we want to integrate with OLM to make delivering software very easy. In our case, we have an extra serviceaccount so we have to let OLM know about this service account so the yaml will be generated properly.
   1. The first thing to do is the modify the Makefile. We need to add extra flags --extra-servive-accounts [see doc](https://sdk.operatorframework.io/docs/advanced-topics/multi-sa/).
   The result look like [this](https://github.com/rocrisp/kubetruth/blob/main/Makefile#L157)
   2. Now we can make the bundle by,
   ````
   make bundle
   ````
   See the directory created by the command [here](https://github.com/rocrisp/kubetruth/tree/main/bundle)

   The manifests directory holds all the generated files from [config/](https://github.com/rocrisp/kubetruth/tree/main/config)

6. The crd, projectmapping.yaml, in helm-chart/kubetruth/crd/ directory will be automatically deployed if you put it [here](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/projectmapping.yaml)
7.  The rbac permission from [here](https://github.com/cloudtruth/kubetruth/blob/981d3719a4e1ab6c70e9f8e6c41ed21da06d3acb/helm/kubetruth/values.yaml#L26) is added to csv [here](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/kubetruth-operator.clusterserviceversion.yaml#L95) and [here](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/kubetruth-operator.clusterserviceversion.yaml#L338)
8.  There are a few environment variables the Makefile depends. I added a setenv.sh file to make it easy for when you need to build, push containers. [see here](https://github.com/rocrisp/kubetruth/blob/main/setenv.sh)
9.  You can execute the setenv.sh with this command,
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