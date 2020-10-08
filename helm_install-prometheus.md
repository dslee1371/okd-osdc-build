## install prometheus package with helm 



## helm
- reference documents : 
	+ Helm install : https://helm.sh/docs/intro/install/
	+ Helm add repo : https://helm.sh/docs/intro/quickstart/
	+ Prometheus : https://github.com/helm/charts/tree/master/stable/prometheus
- install
	+ download link : https://github.com/helm/helm/releases

- Initialize a Helm Chart Repository
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm search repo stable
helm repo update 
```

- Create project
```
oc new-project monitoring
# helm install --name prometheus-ds stable/prometheus
# helm install --name prometheus-ds stable/prometheus
# helm install my-prometheus-operator stable/prometheus-operator
helm install stable/prometheus-operator --generate-name
```

- pvc
```
[root@l1-31-master1 my-prometheus]# oc get pvc
NAME                                 STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-1601888812-alertmanager   Pending                                                      6m
prometheus-1601888812-server         Pending                                                      6m
```

- Uninstall prometheus chart
```
helm delete prometheus-1601888812
```

## helm prometheus install
- reference documents : https://github.com/prometheus-community/helm-charts

- Commands
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm search repo prometheus-community
helm install prometheus-community/prometheus --generate-name
```

## helm charts 
```
git clone https://github.com/helm/charts.git
cd ./charts/stable/prometheus
```

## Labels
```
[root@l1-31-master1 prometheus]# kubectl get nodes --show-labels
NAME                               STATUS    ROLES     AGE       VERSION           LABELS
l1-31-master1.cluster1.dslee.lab   Ready     master    24d       v1.11.0+d4cacc0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=l1-31-master1.cluster1.dslee.lab,node-role.kubernetes.io/master=true
l1-32-master2.cluster1.dslee.lab   Ready     master    24d       v1.11.0+d4cacc0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=l1-32-master2.cluster1.dslee.lab,node-role.kubernetes.io/master=true
l1-33-master3.cluster1.dslee.lab   Ready     master    24d       v1.11.0+d4cacc0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=l1-33-master3.cluster1.dslee.lab,node-role.kubernetes.io/master=true
l1-41-infra1.cluster1.dslee.lab    Ready     infra     24d       v1.11.0+d4cacc0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=l1-41-infra1.cluster1.dslee.lab,node-role.kubernetes.io/infra=true
l1-42-infra2.cluster1.dslee.lab    Ready     infra     24d       v1.11.0+d4cacc0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=l1-42-infra2.cluster1.dslee.lab,node-role.kubernetes.io/infra=true
l1-71-node1.cluster1.dslee.lab     Ready     compute   24d       v1.11.0+d4cacc0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=l1-71-node1.cluster1.dslee.lab,node-role.kubernetes.io/compute=true
l1-72-node2.cluster1.dslee.lab     Ready     compute   24d       v1.11.0+d4cacc0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=l1-72-node2.cluster1.dslee.lab,node-role.kubernetes.io/compute=true
```

`node-role.kubernetes.io/infra: true`

## prometheus deprecated
```
root@l1-31-master1:~/git/charts/stable/prometheus-operator# helm install stable/prometheus-operator --generate-name
WARNING: This chart is deprecated
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
Error: unable to build kubernetes objects from release manifest: error validating "": error validating data: ValidationError(MutatingWebhookConfiguration.webhooks[0].clientConfig): missing required field "caBundle" in io.k8s.api.admissionregistration.v1beta1.WebhookClientConfig
```

## Guide 
```
DEPRECATED
Further development has moved to . The chart has been renamed  to more clearly reflect that it installs the `kube-prometheus project stack`, within which Prometheus Operator is only one componen
```

## retry
```
helm install prometheus-community/kube-prometheus-stack --generate-name
```




