# HA Cockroach DB
OCP 4.9.9 on AWS

## Test 0: OCP Install with CockroachDB, fail nodes
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
kubectl run crdb-tester --image=cockroachdb/cockroach:v21.2.4 --command -- sleep 9999s
```

**Stop 1 Instance in AWS**   
Write Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="INSERT INTO roaches VALUES ('A', 'Apple'), ('B', 'Banana')"
```
output:
```
Error from server: error dialing backend: dial tcp 10.0.153.176:10250: connect: no route to host
```

Read Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="SELECT * FROM roaches;"
```
output:
```
Error from server: error dialing backend: dial tcp 10.0.153.176:10250: connect: no route to host
```

**Stop Another Instance in AWS**   
Write Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="INSERT INTO roaches VALUES ('c', 'Candy'), ('D', 'Donut')"
```
output
```
Error from server: error dialing backend: dial tcp 10.0.153.176:10250: connect: no route to host
```
Read Test
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="SELECT * FROM roaches;"
```
output
```
Error from server: error dialing backend: dial tcp 10.0.153.176:10250: connect: no route to host
```

**Stop Start Instances in AWS**  
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


Time: 29ms
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

Delete pod
```
kubectl delete pod crdb-tester --force --grace-period=0
```
## Test 1: OCP Install with Node Health Check and Poison Pill
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
kubectl run crdb-tester --image=cockroachdb/cockroach:v21.2.4 --command -- sleep 9999s
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

kubectl get pod -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase -n cockroachdb -w