## How to convert KubeTruth K8s Operator to Helm-based Operator using the [Operator SDK - Operator framework](https://sdk.operatorframework.io/docs/building-operators/helm/)

### Our objective is to migrate the KubeTruth Operator, which was initially built using Ruby and Helm Chart, into a Kubernetes Operator that is compatible with Openshift 4.12, while keeping the required changes to the source code to a minimum.

### Initially, our objective is to confirm that KubeTruth operates correctly on Openshift 4.12.

Follow directions [in here](https://docs.cloudtruth.com/integrations/kubernetes) to install KubeTruth on Openshift 4.12.
  
After KubeTruth installed on Openshift 4.12, we see the pod is not running, and it has a Error status.

* This is no longer occurring due to an update in the configuration of the container. Proceed to the section on [Building the operator](https://github.com/rocrisp/kubetruth#building-operator).
  
The Pod's log offers insight into what's happening with the container. 
   
````
$ oc logs kubetruth-install-b8867597b-6cdqq
Starting app
/usr/local/lib/ruby/3.0.0/bundler/shared_helpers.rb:105:in `rescue in filesystem_access': There was an error while trying to write to `/srv/app/Gemfile.lock`. It is likely that you need to grant write permissions for that path.
```` 
So there's a problem writing to /srv/app/Gemfile.lock

However, this is a correct behavior in Openshift.

Openshift uses security context constraints [(SCCs)](https://docs.openshift.com/container-platform/4.12/authentication/managing-security-context-constraints.html#security-context-constraints-about_configuring-internal-oauth) to control permissions for the pods in your cluster.

Openshift put restricted-v2 SCC on pods with the exception of default, kube-system, and openshift-operators namespaces.

````
$ oc get pod kubetruth-install-777d7d8745-xnxhd -oyaml | grep scc
    openshift.io/scc: restricted-v2
````

So now let's examine restricted-v2 :

````
oc describe scc restricted-v2
````
The reason why we can't write to /srv/app/Gemfile.lock is because the pod runs with userid => 1000740000, and the container is using userid smaller.

````
$ oc get pod kubetruth-install-777d7d8745-xnxhd -oyaml | grep runAsUser
      runAsUser: 1000740000
````

To get around this without making changes to the container's permission, we can add anyuid to scc policy for the pod's serviceaccount. Anyuid allowes any userid to read/write to the container.

    oc describe scc anyuid 

1. Get pod's ServiceAccountName

````
$ oc get pod kubetruth-install-b8867597b-6cdqq -oyaml | grep serviceAccountName
  serviceAccountName: kubetruth-install
````

2. Add anyuid to kubetruth-install serviceaccount

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
   This means that there's no restriction on who can read/write containers in the pod.

Having resolved that matter, we can now proceed to construct a Helm-based Operator using the Helm Chart supplied by KubeTruth.

## <a id="BuildingOperator"></a>Building Operator

[For reference, this is the KubeTruth Operator repo](https://github.com/cloudtruth/kubetruth)

#### Prior to starting, there are some factors to take into account.

1. As the Ruby Operator already has a custom Kind associated with it, we'll need to create a new Kind with a distinct name.
   
2. Our goal is to remove any rbac and serviceaccount resources from the helm charts, as OLM manages all rbac resources.
   
~~3. It is necessary to include anyuid in the clusterrole and then assign it to the serviceaccount we'll be generating.~~


#### Create a new project.

1. Create a Helm project called "kubetruth-operator"
````
mkdir kubetruth-operator
operator-sdk init --domain cloudtruth.com --plugins helm
````
2. Create the API.
   
   This creates the kubetruth-operator project for watching the KubeTruth resource.

````
operator-sdk create api \
 --group=apps --version=v1alpha1 \
 --kind=KubeTruth \
 --helm-chart=/Users/rosecrisp/codebase/kubetruth/helm/kubetruth
````
   Note: Fetch the Helm chart from my local disk

2. Rbac and serviceAccount will be handled by OLM so we need to do the following :
    
    1. Extract rbac contents from [helm-charts/kubetruth/values.yaml](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/values.yaml#L26) and add them to config/rbac/role.yaml. [See Example](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/role.yaml#L83)
   
    2. Since we'll be generating a new ServiceAccount, please delete serviceaccount from [helm-charts/kubetruth/values.yaml](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/values.yaml#L17)
    
   
    3. To enable OLM, extract clusterrole.yaml, clusterrolebinding.yaml, role.yaml, rolebinding.yaml, and serviceaccount.yaml from helm-charts/kubetruth/templates dir (https://github.com/rocrisp/kubetruth/tree/main/helm-charts/kubetruth/templates)
   
    4. and add additional_serviceaccount.yaml, additional_clusterrole.yaml, and additional_clusterrole_binding.yaml files in the config/rbac directory.[See example](https://github.com/rocrisp/kubetruth/tree/main/config/rbac)

    5. Modify config/rbac/kustomization.yaml to include the files created in (iv). [See example](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/kustomization.yaml#L20)

    ~~Note: Notice the clusterrole defines SCC with anyuid which allows anyuid to access the container in the pod [See example](https://github.com/rocrisp/kubetruth/blob/main/config/rbac/kubetruth_install_clusterrole.yaml#L41)~~

3.  Move projectmapping.yaml from helm-chart/kubetruth/crd dir to config/crd/bases/projectmapping.yaml. [See example](https://github.com/rocrisp/kubetruth/tree/main/config/crd/bases)
4.  Modify config/crd/kustomization.yaml to include projectmapping.yaml. [See example](https://github.com/rocrisp/kubetruth/blob/main/config/crd/kustomization.yaml#L6)
5.  Add a sample cr for ProjectMapping. [See example](https://github.com/rocrisp/kubetruth/blob/main/config/samples/apps_v1alpha1_projectmapping.yaml)
6. To use the new serviceaccount generated by OLM, modify the serviceAccountName field in helm-charts/kubetruth/templates/deployment.yaml. to "kubetruth-operator-additional-service-account" [See example](https://github.com/rocrisp/kubetruth/blob/main/helm-charts/kubetruth/templates/deployment.yaml#L27)
7. To maximize the benefits of the Makefile, adjust the Makefile to set up the image registry for the Operator and version
   ````
   VERSION ?= 0.0.1
   IMAGE_TAG_BASE ?= quay.io/placeholder/kubetruth
   IMG ?= $(IMAGE_TAG_BASE):$(VERSION)
   ````

8. Once completed, the subsequent command will build and upload an operator image labeled as quay.io/placeholder/kubetruth-operator:v0.0.1.
   ````
   make docker-build docker-push
   ````
9.  Associate the serviceaccount with OLM. 

   * Add extra flag --extra-servive-accounts to the Makefile [see doc](https://sdk.operatorframework.io/docs/advanced-topics/multi-sa/).
        
        [See example](https://github.com/rocrisp/kubetruth/blob/main/Makefile#L162)
   
   * The subsequent command generates a bundle that OLM can utilize to install the operator. [Bundle Documentation](https://sdk.operatorframework.io/docs/olm-integration/generation/), so the following command bundles your operator
      ````
      make bundle
      ````
     [See example](https://github.com/rocrisp/kubetruth/tree/main/bundle)
   
   * Build and push the Operator bundle image
     ````
     make bundle-build bundle-push
     ````
   * As the additional serviceAccount is created in  kubetruth namespace, [see example](https://github.com/rocrisp/kubetruth/blob/main/bundle/manifests/kubetruth-operator-extra-clusterrolebinding_rbac.authorization.k8s.io_v1_clusterrolebinding.yaml#L13), we must install the operator in the kubetruth namespace.

     NOTE: You can modify the namespace by editing [this line](https://github.com/rocrisp/kubetruth/blob/main/config/default/kustomization.yaml#L2).

### Deploy the Kubetruth operator in the cluster
 *   Generate a namespace named kubetruth
      ````
      oc new-project kubetruth
      ```` 
    
 *  Install the operator utilizing Operator SDK integration with OLM.
     ````
     operator-sdk run bundle quay.io/rocrisp/kubetruth-operator-bundle:v3.0.0
     ````
    
 *  Install the operand
    ````
    oc apply -f config/samples/apps_v1alpha1_kubetruth.yaml
    ````

### End

### Freebies

This command will delete the resources generated by OLM for the kubetruth-operator installation
````
operator-sdk cleanup kubetruth-operator
````
Additionally, on occasions where you need to edit the CSV file, this command verifies the syntax to ensure that all details are correct
````
operator-sdk bundle validate ./bundle
````