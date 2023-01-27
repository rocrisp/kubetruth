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
1. Rbac and serviceAccount will be handled by OLM so we need to remove the following :
    
    1. Extract rbac contents from helm-charts/kubetruth/values.yaml and add them to config/rbac/role.yaml. [See Example](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/role.yaml)
   
    2. Remove serviceAccount contents from helm-charts/kubetruth/values.yaml.
   
    [See example](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/values.yaml) 
   
    3. Remove clusterrole.yaml, clusterrolebinding.yaml, role.yaml, rolebinding.yaml, and serviceaccount.yaml from helm-charts/kubetruth/templates dir [See example](https://github.com/rocrisp/kubetruth/tree/main/helm-charts/kubetruth/templates)
   
    4. Create serviceaccount.yaml, clusterrole.yaml, and clusterrolebinding.yaml in the config/rbac/ subdir [See example](https://github.com/rocrisp/kubetruth/tree/main/config/rbac)

    5. Modify config/rbac/kustomization.yaml to include the files created in (iv). [See example](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/kustomization.yaml#L20)

    Note: Notice the clusterrole defines SCC with anyuid which allows anyuid to access the container in the pod [See example](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/kubetruth_install_clusterrole.yaml#L41)

2.  Move projectmapping.yaml from helm-chart/kubetruth/crd/ directory to config/crd/bases/projectmapping.yaml.[See example](https://github.com/rocrisp/kubetruth/tree/main/config/crd/bases)
3.  Modify config/crd/kustomization.yaml to include projectmapping.yaml. [See example](https://github.com/rocrisp/kubetruth/blob/main/config/crd/kustomization.yaml#L6)
4.  Add a sample cr for kind: ProjectMapping. [See example](https://github.com/rocrisp/kubetruth/blob/main/config/samples/apps_v1alpha1_projectmapping.yaml)
5. Update serviceAccountName in helm-charts/kubetruth/templates/deployment.yaml to "kubetruth-operator-kubetruth-install" [See example](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/templates/deployment.yaml#L27)
6. Integrate with OLM to make delivering software very easy. 

   1. Add the extra serviceaccount to OLM by adding an extra flag --extra-servive-accounts to the Makefile [see doc](https://sdk.operatorframework.io/docs/advanced-topics/multi-sa/).
        
        [See example](https://github.com/rocrisp/kubetruth/blob/main/Makefile#L157)
   
   2. Makefile depends on environment variables. To make things easier use setenv.sh. [See example](https://github.com/rocrisp/kubetruth/blob/main/setenv.sh)
   3.  Modify and execute setenv.sh with
    ````
    source setenv.sh
    ````

   4. OLM understand files in bundle format. [Bundle Documentation](https://sdk.operatorframework.io/docs/olm-integration/generation/)
   
    ````
    make bundle
    ````
    [See example](https://github.com/rocrisp/kubetruth/tree/main/bundle)


7.   Build and push the Operator, and the Operator bundle
````
make docker-build docker-push
make bundle-build bundle-push
````

8.   The extra serviceAccount is created in  kubetruth-operator-system namespace. [See example](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/kubetruth-operator-kubetruth-install-clusterrolebinding_rbac.authorization.k8s.io_v1_clusterrolebinding.yaml#L13)

NOTE: You can change the namespace that's hardwired in the manifest yaml file.

Create the namespace

    oc new-project kubetruth-operator-system
    
    
9.   Deploy the operator using Operator SDK intergration with OLM
   
    operator-sdk run bundle quay.io/rocrisp/kubetruth-operator-bundle:v3.0.0
    
10.  Deploy the operand
````
oc apply -f config/samples/apps_v1alpha1_kubetruth.yaml
````

This is a handy command for when you're installing and deleting operators .

It will remove resources created by OLM.
````
operator-sdk cleanup kubetruth-operator
````

Also, there are times you need to modify CSV file. This command validates the syntax to make sure dot the i's and cross the t's.
````
operator-sdk bundle validate ./bundle
````