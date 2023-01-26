oc new-project kubetruth-operator-system

operator-sdk run bundle quay.io/rocrisp/kubetruth-operator-bundle:v0.1.0

oc apply -f config/samples/apps_v1alpha1_kubetruth.yaml