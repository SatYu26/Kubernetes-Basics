# The Kubecon + CloudNativeCon Special Edition Lab

The various parts of the Kubernetes Control Plane, such as the kube-apiserver and kubelet processes, govern how Kubernetes communicates with your cluster. The Control Plane maintains a record of all of the Kubernetes Objects in the system, and runs continuous control loops to manage those objects’ state. At any given time, the Control Plane’s control loops will respond to changes in the cluster and work to make the actual state of all the objects in the system match the desired state that you provided.
<img src="images\control-plane.png">
For example, when you use the Kubernetes API to create a Deployment object, you provide a new desired state for the system. The Kubernetes Control Plane records that object creation, and carries out your instructions by starting the required applications and scheduling them to cluster nodes–thus making the cluster’s actual state match the desired state.

## Kubernetes Control Plane

The Kubernetes Control Plane is responsible for maintaining the desired state for your cluster. When you interact with Kubernetes, such as by using the kubectl command-line interface, you’re communicating with your cluster’s Kubernetes Control Plane.

The “Control Plane” refers to a collection of processes managing the cluster state. Typically these processes are all run on a single node in the cluster, and this node is also referred to as the Control Plane. The Control Plane can also be replicated for availability and redundancy.

## Kubernetes Nodes

The nodes in a cluster are the machines (VMs, physical servers, etc) that run your applications and cloud workflows. The Kubernetes Control Plane controls each node; you’ll rarely interact with nodes directly.
<img src="images\nodes.png">

Kubernetes contains a number of abstractions that represent the state of your system: deployed containerized applications and workloads, their associated network and disk resources, and other information about what your cluster is doing. These abstractions are represented by objects in the Kubernetes API.

A Pod is the basic building block of Kubernetes–the smallest and simplest unit in the Kubernetes object model that you create or deploy. A Pod represents a running process on your cluster.

A Pod encapsulates an application container (or, in some cases, multiple containers), storage resources, a unique network IP, and options that govern how the container(s) should run. A Pod represents a unit of deployment: a single instance of an application in Kubernetes, which might consist of either a single container or a small number of containers that are tightly coupled and that share resources.

Pods in a Kubernetes cluster can be used in two main ways:

- Pods that run a single container. The “one-container-per-Pod” model is the most common Kubernetes use case; in this case, you can think of a Pod as a wrapper around a single container, and Kubernetes manages the Pods rather than the containers directly.

- Pods that run multiple containers that need to work together. A Pod might encapsulate an application composed of multiple co-located containers that are tightly coupled and need to share resources. These co-located containers might form a single cohesive unit of serviceone container serving files from a shared volume to the public, while a separate “sidecar” container refreshes or updates those files. The Pod wraps these containers and storage resources together as a single manageable entity.

Each Pod is meant to run a single instance of a given application. If you want to scale your application horizontally (e.g., run multiple instances), you should use multiple Pods, one for each instance.

In Kubernetes, this is generally referred to as replication. Replicated Pods are usually created and managed as a group by an abstraction called a Controller, for example a ReplicaSet controller (to be discussed) maintains the Pod lifecycle. This includes Pod creation, upgrade and deletion, and scaling.

Containers in a pod run in the same Network namespace, so they share the same IP address and port space.

All the containers in a pod also have the same loopback network interface, so a container can communicate with other containers in the same pod through localhost.
<img src="images\pods.svg">

A **Deployment** controller provides declarative updates for Pods and ReplicaSets.

You describe a desired state in a Deployment object, and the Deployment controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.

Deployment manages the ReplicaSet to orchestrate Pod lifecycles. This includes Pod creation, upgrade and deletion, and scaling.

# Getting Started

**Introduction to K10**

Kasten’s K10 data management platform, purpose-built for Kubernetes, provides enterprise operations teams an easy-to-use, scalable, and secure system for backup and restore, disaster recovery, and mobility of Kubernetes applications.

K10’s application-centric approach and deep integrations with relational and NoSQL databases, Kubernetes distributions, and all clouds provides teams the freedom of infrastructure choice without sacrificing operational simplicity. Policy-driven and extensible, K10 provides a native Kubernetes API and includes features such full-spectrum consistency, database integrations, automatic application discovery, multi-cloud mobility, and a powerful web-based user interface.

For the purpose of this test-drive, we have installed Kasten K10 and created a MySQL database k10demo in Kubernetes.

- In the terminal window, run the following commands to verify the database was created:<br>

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SHOW DATABASES LIKE 'k10demo'"
```

We will now create a policy to backup MySQL.

- From the main Dashboard tab, click on the Policies card. There should be no policies visible at this point.

- - Click Create New Policy and:

- Give the policy the name backup-mysql
- Select Snapshot for the Action
- Select Hourly for the Action Frequency
- Leave the Snapshot Retention selection as-is
- Select By Name for Select Applications and then, from the dropdown, select mysql
- Leave all other settings as-is and select Create Policy

- **Running the Policy**

The above policies will only run at the scheduled time (by default, at the top of the hour). To run the policies manually for the first time, click on Run Once on the Policies page. Confirm by clicking Run Policy and then go back to the main dashboard to view the job in action. Verify that the job has successfully completed.

Now that we have a MySQL backup, let's go simulate accidental data loss and then recover the system from that loss.

- **Causing Data Loss**
- - For the purposes of this test drive, we will simply drop the k10demo database we had created earlier.

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "DROP DATABASE k10demo"
```

- Verify that the database has been deleted by running:<br>
  `kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SHOW DATABASES LIKE 'k10demo'"`

- **Recovering Data**
  To recover the databases, go to the K10 dashboard, click Applications, and then select Restore on the MySQL card.

Click on a recent restore point and then select the Exported restore point (this is stored in the object storage system instead of as a non-durable snapshot on the storage system). In this case, we will select the default Application Name option to restore in place (Restore as "mysql"). Leave all other selections as-is, click on Restore, and confirm the action.

Return to the main dashboard to view the Restore job and verify that it completes successfully.

- To verify that our data was recovered, run the following command in the terminal to view the restored database:

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SHOW DATABASES LIKE 'k10demo'"
```
