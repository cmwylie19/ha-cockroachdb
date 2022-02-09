# HA Cockroach DB
OCP 4.9.9 on AWS
Based on [HA Cockroach DB](https://docs.google.com/document/d/1d4W69xKwLtjhteCi604jl33EDGVaEJJ_2zmzIgeNfuw/edit)

### TOC
- [Only CockroachDB](#test-0)
- [CockroachDB, NodeHealthCheck, PoisonPill](#test-1)
- [CockroachDB, MachineHealthCheck, PoisonPill](#test-2)
- [Commands Used](#commands)

## Considerations
Since this test involves stopping instances in AWS, we have to be confident that the pod that we are sending requests from is not on an instance that goes down. Due to this, we will be deployering the `crdb-tester` pod with tolerations and a nodeSelector to schedule it on the master node.
## Test 0: 
### OCP Install with CockroachDB, fail nodes
_Scenerio_: When we deploy a StatefulSet of 3 replicas and stop a node are we still able to read and write from the database?

_Scenerio_: When we deploy a StatefulSet of 3 replicas and stop two nodes are we still able to read and write from the database?

### _Setup_:   
Install CockroachDB
```
helm install cockroachdb -n cockroachdb charts/cockroachdb
```

Insert data into CockroachDB
```
kubectl exec -it pod/cockroachdb-0 -n cockroachdb -c cockroachdb -- cockroach sql --insecure --execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')"
```

Create a pod to communicate to the cockroach service
```
helm install crdb-tester charts/tester-pod
```

**Stop 1 Instance in AWS**   
Write Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="INSERT INTO roaches VALUES ('A', 'Apple'), ('B', 'Banana')"
```
output:
```
INSERT 2


Time: 860ms
```

Read Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="SELECT * FROM roaches;"
```
output:
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
  A                     | Apple
  B                     | Banana
(4 rows)


Time: 7ms
```

**Stop Another Instance in AWS**   
Write Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="INSERT INTO roaches VALUES ('c', 'Candy'), ('D', 'Donut')"
```
output
```
ERROR: cannot dial server.
Is the server running?
If the server is running, check --host client-side and --advertise server-side.

dial tcp 172.30.252.79:26257: connect: no route to host
Failed running "sql"
command terminated with exit code 1
```
Read Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="SELECT * FROM roaches;"
```
output
```
ERROR: cannot dial server.
Is the server running?
If the server is running, check --host client-side and --advertise server-side.

dial tcp 172.30.252.79:26257: connect: no route to host
Failed running "sql"
command terminated with exit code 1
```

### _Cleanup_:   
Install CockroachDB
```
helm uninstall cockroachdb -n cockroachdb 
```

Make sure everything is gone
```
kubectl get pod,pvc -n cockroachdb 
kubectl delete pods,pvc --all -n cockroachdb
```

Delete tester pod
```
helm uninstall crdb-tester
```
## Test 1
### OCP Install with Node Health Check and Poison Pill
_Scenerio_: When the node is lost poison pill remediation is created and marks the node as scheduling disabled.  
   
### _Setup_:   
Install CockroachDB
```
helm install cockroachdb -n cockroachdb charts/cockroachdb
```

Insert data into CockroachDB
```
kubectl exec -it pod/cockroachdb-0 -n cockroachdb -c cockroachdb -- cockroach sql --insecure --execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')"
```

Install NodeHealthCheck and PoisonPill 
```
helm install nhc -n openshift-operators charts/nhc-operator 
```

Install Node Health Check and Poison Pill Configration
```
helm install node-health-check -n openshift-operators charts/node-health-check
```

Create a pod to communicate to the cockroach service
```
helm install crdb-tester charts/tester-pod
```

**Stop Instances in AWS**  
Write Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="INSERT INTO roaches VALUES ('c', 'Candy'), ('D', 'Donut')"
```
output
```
INSERT 2


Time: 22ms
```
Read Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="SELECT * FROM roaches;"
```
output
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
(2 rows)


Time: 2ms
```

### _Cleanup_:   
Install CockroachDB
```
helm uninstall cockroachdb -n cockroachdb 
```

Make sure everything is gone
```
kubectl get pod,pvc -n cockroachdb 

kubectl delete pod,pvc -n cockroachdb --all
```
Uninstall
Install Node Health Check and Poison Pill Configration
```
helm uninstall node-health-check -n openshift-operators
```

Uninstall NodeHealthCheck and PoisonPill
```
helm uninstall nhc -n openshift-operators 
```


## Test 2
### OCP Install with Machine Health Check and Poison Pill
_Scenerio_: A node is not able to reach the api server due to transient failure. The poison pill remediation marked the node as SchedulingDisabled. But once the node was back, the node was eventually marked as Ready. The pod was rerun in the same node.  
   
### _Setup_:   
Install CockroachDB
```
helm install cockroachdb -n cockroachdb charts/cockroachdb
```

Insert data into CockroachDB
```
kubectl exec -it pod/cockroachdb-0 -n cockroachdb -c cockroachdb -- cockroach sql --insecure --execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')"
```

Install MachineHealthCheck and PoisonPill 
```
helm install mch -n openshift-operators charts/machine-health-check 
```

Create a pod to communicate to the cockroach service
```
helm install crdb-tester charts/tester-pod
```

**Stop Instance in AWS**  
Write Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="INSERT INTO roaches VALUES ('c', 'Candy'), ('D', 'Donut')"
```
output
```
INSERT 2


Time: 22ms
```
Read Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="SELECT * FROM roaches;"
```
output
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
(2 rows)


Time: 2ms
```

### _Cleanup_:   
Install CockroachDB
```
helm uninstall cockroachdb -n cockroachdb 
```

Make sure everything is gone
```
kubectl get pod,pvc -n cockroachdb 

kubectl delete pod,pvc -n cockroachdb --all
```

Uninstall MachineHealthCheck and PoisonPill
```
helm uninstall mch -n openshift-operators 
```

Delete pod
```
helm uninstall crdb-tester
```

## Commands
Watch pod name, node of pod, and pod status in `cockroachdb`
```
kubectl get pod -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase -n cockroachdb -w
```

Watch PoisonPillRemediation in all namespaces
```
kubectl get ppr -A -w
```
Watch nodes in `cockroachdb`
```
kubectl get nodes -n cockroachdb -w
```