# OKD install issue logs 

## Ansible failed 
```
[issue]

[root@l1-31-master1 ~]# docker pull registry.access.redhat.com/openshift3/registry-console:v3.11
Trying to pull repository registry.access.redhat.com/openshift3/registry-console ...
open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory


[solution]

$ sudo -i && cd /tmp $ wget http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm
$ rpm2cpio python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm | cpio -iv --to-stdout ./etc/rhsm/ca/redhat-uep.pem | tee /etc/rhsm/ca/redhat-uep.pem
```

## Ansible failed 2

```
[issue]
INSTALLER STATUS ***************************************************************************************************************************************************************************************************************************
Initialization               : Complete (0:00:12)
Health Check                 : Complete (0:00:38)
Node Bootstrap Preparation   : Complete (1:31:42)
etcd Install                 : Complete (0:00:19)
NFS Install                  : Complete (0:00:03)
Load Balancer Install        : Complete (0:00:17)
Master Install               : Complete (0:03:54)
Master Additional Install    : Complete (0:00:19)
Node Join                    : Complete (0:00:32)
Hosted Install               : Complete (0:00:26)
Cluster Monitoring Operator  : Complete (0:00:36)
Web Console Install          : Complete (0:00:36)
Console Install              : Complete (0:00:26)
Metrics Install              : In Progress (0:00:02)
        This phase can be restarted by running: playbooks/openshift-metrics/config.yml
Friday 11 September 2020  11:34:38 +0900 (0:00:00.022)       1:40:09.578 ******
===============================================================================
openshift_node : Install node, clients, and conntrack packages ------------------------------------------------------------------------------------------------------------------------------------------------------------------- 5448.33s
openshift_control_plane : Wait for all control plane pods to come up and become ready --------------------------------------------------------------------------------------------------------------------------------------------- 122.86s
Run health checks (install) - EL --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 38.10s
openshift_web_console : Verify that the console is running ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 31.12s
openshift_cluster_monitoring_operator : Wait for the ServiceMonitor CRD to be created ---------------------------------------------------------------------------------------------------------------------------------------------- 30.54s
openshift_console : Waiting for console rollout to complete ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 21.87s
openshift_loadbalancer : Install haproxy ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 12.60s
openshift_ca : Install the base package for admin tooling -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 12.28s
openshift_manage_node : Wait for sync DS to set annotations on all nodes ----------------------------------------------------------------------------------------------------------------------------------------------------------- 11.26s
openshift_node_group : Wait for the sync daemonset to become ready and available --------------------------------------------------------------------------------------------------------------------------------------------------- 10.55s
openshift_hosted : Create OpenShift router ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 6.83s
Approve node certificates when bootstrapping ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 5.88s
openshift_manageiq : Configure role/user permissions -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 4.02s
openshift_excluder : Install docker excluder - yum ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 3.83s
openshift_excluder : Install openshift excluder - yum ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 3.66s
openshift_master_certificates : copy ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 3.37s
openshift_control_plane : Wait for APIs to become available ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.79s
openshift_hosted : Create default projects ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 2.20s
Restart tuned service --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.12s
openshift_hosted : Add the secret to the registry's pod service accounts ------------------------------------------------------------------------------------------------------------------------------------------------------------ 1.50s


Failure summary:


  1. Hosts:    l1-31-master1.cluster1.dslee.lab
     Play:     OpenShift Metrics
     Task:     openshift_metrics : fail
     Message:  'keytool' is unavailable. Please install java-1.8.0-openjdk-headless on the control node


[solution]
yum -y install java-1.8.0-openjdk-headless # for boot,all master
```

## Ansible failed #3

```
[issue]
 "msg": "Failed to import the required Python library (passlib) 

[solution]
yum install python-passlib -y # for boot server
```

## Firewall issue #1

```
[issue]
[root@l1-31-master1 ~]# showmount -e l1-03-nfs1.cluster1.dslee.lab
clnt_create: RPC: Port mapper failure - Unable to receive: errno 113 (No route to host)

[solution]
rpcinfo -p
iptables -A INPUT -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -p udp --dport 111 -j ACCEPT
iptables -A INPUT -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -p udp --dport 2049 -j ACCEPT
iptables -A INPUT -p tcp --dport 20048 -j ACCEPT
# setenforce 0
# sestatus 

service iptables save
iptables-save > /etc/sysconfig/iptables

iptables -A OUTPUT -p tcp --dport 2049 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --sport 2049 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p udp --dport 2049 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p udp --sport 2049 -m state --state ESTABLISHED -j ACCEPT


# edit /etc/sysconfig/nfs
LOCKD_TCPPORT=32803
# UDP port rpc.lockd should listen on.
LOCKD_UDPPORT=32769
MOUNTD_PORT=892

# restart service
systemctl restart rpcbind
systemctl restart nfs
iptables -A INPUT -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -p udp --dport 892 -j ACCEPT
service iptables save
iptables-save > /etc/sysconfig/iptables
```

## stuck delete project
```

[Issue]
Terminations

[Solution]
oc get project test-o json > ns-without-finalizers.json

kubectl proxy & PID=$!
curl -X PUT http://localhost:8001/api/v1/namespaces/test/finalize \
-H "Content-Type: application/json" --data-binary @ns-without-finalizers.json
kill $PID

kubectl patch ns test -p '{"metadata":{"finalizers":null}}'

kubectl get namespace test -o json > logging.json
kubectl replace --raw "/api/v1/namespaces/ns/finalize" -f ./logging.json
```

## failed log in

```
[Issue]
Error from server (InternalError): Internal error occurred: unexpected response: 400 on master server 

[solution]
# sync /etc/origin/master/htpasswd file
# master-restart api
# master-restart controllers

htpasswd -c -b /etc/origin/ma

oc login --loglevel=10
```

## Logging ansible failed 

```
[Issue - enable logging]

TASK [Run variable sanity checks] ********************************************************************************************************************************************
fatal: [l1-31-master1.cluster1.dslee.lab]: FAILED! => {"msg": "last_checked_host: l1-31-master1.cluster1.dslee.lab, last_checked_var: openshift_logging_storage_kind;nfs is an unsupported type for openshift_logging_storage_kind. openshift_enable_unsupported_configurations=True must be specified to continue with this configuration."}

-- solution
openshift_enable_unsupported_configurations=True
```

## Logging ansible failed

```
[Issue - enable loggin ]

TASK [assert] ****************************************************************************************************************************************************************
fatal: [l1-31-master1.cluster1.dslee.lab]: FAILED! => {
    "assertion": "openshift_logging_es_nodeselector is defined",
    "changed": false,
    "evaluated_to": false,
    "msg": "A node selector is required for Elasticsearch pods, please specify one with openshift_logging_es_nodeselector"

[solution]
node-role.kubernetes.io/infra=true
> openshift_logging_es_nodeselector={"node-role.kubernetes.io/infra":"true"}
```

## Hyper-v issue logs

```
[Issue]
stuck stop vm

[solutions]
process stop vmwp.exe < no effort, restart vm, power off
process stop vmms.exe < entire 
```
