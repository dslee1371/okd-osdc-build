# install commands

##  configure htpasswd
``` 
htpasswd -cb /etc/origin/master/htpasswd ocpadmin test1234!
htpasswd -cb /etc/origin/master/htpasswd ocpadmin test1234!
htpasswd /etc/origin/master/htpasswd admin

export KUBECONFIG=/home/user_id/install/auth/kubeconfig
oc login
```


## Native high availability cluster method with optional load balancer.
 
```
ansible-playbook -i /etc/ansible/hosts playbooks/deploy_cluster.yml
ansible-playbook -i /etc/ansible/hosts playbooks/adhoc/uninstall.yml
```


## Uninstall openshift monitoring
```
$ ansible-playbook [-i </path/to/inventory>] <OPENSHIFT_ANSIBLE_DIR>/playbooks/openshift-monitoring/config.yml \
   -e openshift_cluster_monitoring_operator_install=False

ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/openshift-monitoring/config.yml -e openshift_cluster_monitoring_operator_install=False
```

## Install metric
```
- Install

ansible-playbook -i /etc/ansible/hosts-metric /usr/share/ansible/openshift-ansible/playbooks/openshift-metrics/config.yml <-- error

ansible-playbook -i /etc/ansible/hosts-metric /usr/share/ansible/openshift-ansible/playbooks/openshift-metrics/config.yml \
   -e openshift_metrics_install_metrics=True \
   -e openshift_metrics_cassandra_storage_type=pv

- Uninstall

ansible-playbook -i /etc/ansible/hosts-metric /usr/share/ansible/openshift-ansible/playbooks/openshift-metrics/config.yml \
   -e openshift_metrics_install_metrics=False 
```

## Install monitor for openshift
```
ansible-playbook -i /etc/ansible/hosts-monitor /usr/share/ansible/openshift-ansible/playbooks/openshift-monitoring/config.yml \
   -e openshift_cluster_monitoring_operator_install=True
```

## permission
```
oc adm policy add-cluster-role-to-user cluster-admin ocpadmin
oc adm policy add-role-to-user admin ocpadmin -n openshift
oc adm policy add-role-to-user view system:serviceaccount:default:default -n default

oc adm policy add-role-to-user system:registry ocpadmin
oc adm policy add-role-to-user system:image-builder ocpadmin

oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:default:router

# Grant all authenticated users access to the anyuid SCC
oc adm policy add-scc-to-user anyuid -n default -z router
oc adm policy add-scc-to-group anyuid system:authenticated
```

## install logging
```
ansible-playbook -i /etc/ansible/hosts-logging /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml -e openshift_logging_install_logging=true
ansible-playbook -i /etc/ansible/hosts-logging-v0.1 /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml
edit host inventory
ansible-playbook -i /etc/ansible/hosts-logging-v0.2 /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml -e openshift_logging_install_logging=false
ansible-playbook -i /etc/ansible/hosts-logging-v0.1 /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml 
ansible-playbook -i /etc/ansible/hosts-logging-v0.2 /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml 
ansible-playbook -i /etc/ansible/hosts-logging-v0.2 /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml -e openshift_logging_install_logging=false

edit hosts-logging-v0.2
ansible-playbook -i /etc/ansible/hosts-logging-v0.2 /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml 
```
