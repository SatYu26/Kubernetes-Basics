# Lab 2: Build a Kubernetes Application

**Containers** allow you to package your application and its dependencies together into one succinct manifest that can be version controlled, allowing for easy replication of your application across developers on your team and machines in your cluster.
Containers work best for service based architectures. As opposed to monolithic architectures, where every piece of the application is intertwined — from IO to data processing to rendering — service based architectures separate these into separate components.
From an application point of view, instantiating an image (creating a container) is similar to instantiating a process like a service or a web app.

A **stateless app** is an application program that does not save client data generated in one session for use in the next session with that client e.g. print services, microservices.
In **stateful applications**, the state is recorded. By state, we mean any changeable occurrence that includes internal operations, interactions with other applications, environment variables, user-set preferences, memory content, and temporary storage. The data that such applications store depends on their types and other factors under which they operate. Usually, a stateful application is able to record your preferences, track your window size and location, and memorize the files that you have recently opened. Some known examples of stateful applications include MongoDB, Cassandra, and MySQL.

A **Container Image**, in its simplest definition, is a file which is pulled down from a Registry Server and used locally as a mount point when starting Containers. A container is the runtime instantiation of an image.
A **Container Engine** is a piece of software that accepts user requests, including command line options, pulls images, and from the end user's perspective runs the container.

There are many container engines, including Docker, RKT, and CRI-O that provide the runtime environment for the applications inside the container.

You can run them locally or on a remote server.

**Docker** containers that run on Docker Engine are lightweight, portable and secure. They run everywhere: Linux, Windows, Data center, Cloud, Serverless, etc.

Docker Build is at the core of what makes Docker so popular. It allows you to easily create and share portable Docker container images using open standards and create images for multiple CPU and OS architectures and share them in your private registry or on Docker Hub.

Docker Desktop is an application for MacOS and Windows machines for the building and sharing of containerized applications and microservices.
Docker Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.

**Helm** is the package manager for Kubernetes that allows you to package, share and manage the lifecycle of your Kubernetes containerized applications. Essentially you create structured application packages that contain everything they need to run on a Kubernetes cluster; including dependencies the application requires.

Helm uses Charts to pack all the required K8S components for an application to deploy, run and scale. A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might be used to deploy something simple, like a memcached pod, or something complex, like a full web app stack with HTTP servers, databases, caches, and so on.

Helm Templates is subdirectory in a chart that combines the K8S components of it, e.g. Service, ReplicaSet, Deployment, Ingress etc.
Helm Values are described in the values.yaml file which allows users to deploy their containerized applications dynamically. Its Yaml structure holds values that matches the templates defined in the application manifest.

# Getting Started

### Storage Classes

A StorageClass provides a way for administrators to describe the "classes" of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called "profiles" in other storage systems.

- Check the current storage classes created in the cluster:<br>
  `kubectl get storageclasses`

### Persistent Volumes

A PersistentVolume can be mounted on a host in any way supported by the resource provider.

Providers will have different capabilities and each PV's access modes are set to the specific modes supported by that particular volume.

For example, NFS can support multiple read/write clients, but a specific NFS PV might be exported on the server as read-only.

Each PV gets its own set of access modes describing that specific PV's capabilities.
The access modes are:<br>

- ReadWriteOnce -- the volume can be mounted as read-write by a single node
- ReadOnlyMany -- the volume can be mounted read-only by many nodes
- ReadWriteMany -- the volume can be mounted as read-write by many nodes

We are going to create a PersistentVolume of 10Gi, called 'myvolume'.

Make it have accessMode of 'ReadWriteOnce' and 'ReadWriteMany', storageClassName 'local-path', mounted on hostPath '/etc/foo'.

A hostPath PersistentVolume uses a file or directory on the Node to emulate network-attached storage.

- Let's create the directory that we are going to use in this example:<br>
  `mkdir /etc/foo`
- Next, we create the PV:

```
cat<<'EOF' > pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: myvolume
spec:
  storageClassName: local-path
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  hostPath:
    path: /etc/foo
EOF
```

- To create resources in Kubernetes from YAML specification files we use the kubectl apply command:<br>
  `kubectl apply -f pv.yaml`

- Show the PersistentVolumes that exist on the cluster:<br>
  `kubectl get pv`

### Persistent Volume Claims

Pods cannot access Persistent Volumes directly, we need to claim storage capacity for our applications by binding the request for capacity to PVs using PersistentVolumeClaims.

We are going to create a PersistentVolumeClaim, called 'mypvc', with a request of 4Gi and an accessMode of ReadWriteOnce.

```
cat<<'EOF' > pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mypvc
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
EOF
```

- Let's create the PVC:<br>
  `kubectl apply -f pvc.yaml`

- Show the PersistentVolumeClaims of the cluster:<br>
  `kubectl get pvc`

Notice the claim will remain in Pending status until it's consumed by an application. This is due to volumeBindingMode set to WaitForFirstConsumer in the storage class 'local-path' used by the PVC.

Next, let's create a Pod that consumes the claim we created.

```
cat<<'EOF'> busybox-one.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: busybox-one
  name: busybox-one
spec:

  volumes:
  - name:  my-vol # has to match volumeMounts.name
    persistentVolumeClaim:
      claimName: mypvc

  containers:
  - args:
    - /bin/sh
    - -c
    - sleep 3600
    image: busybox
    name: busybox-one

    volumeMounts:
    - name: my-vol # has to match volumes.name
      mountPath: /etc/foo
EOF
```

- Let's create the Pod:<br>
  `kubectl apply -f busybox-one.yaml`

- Next, connect to the pod and copy '/etc/passwd' to '/etc/foo/passwd':<br>
  `kubectl exec busybox-one -it -- cp /etc/passwd /etc/foo/passwd`

Since '/etc/foo' is mounted on the volume we created earlier, it's lifecycle is independant from the Pod's lifecycle. We can attach the volume to another Pod and we should be able to access and read the file we copied in the previous step.

- Let's create a second Pod that mounts to the same PV:

```
cat<<'EOF'> busybox-two.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: busybox-two
  name: busybox-two
spec:

  volumes:
  - name:  my-vol
    persistentVolumeClaim:
      claimName: mypvc

  containers:
  - args:
    - /bin/sh
    - -c
    - sleep 3600
    image: busybox
    name: busybox-two

    volumeMounts:
    - name: my-vol
      mountPath: /etc/foo
EOF
```

Create the Pod<br>
`kubectl apply -f busybox-two.yaml`

Let's connect to it and verify that '/etc/foo' contains the 'passwd' file:<br>
`kubectl exec busybox-two -- ls /etc/foo`

You should be able to see the passwd file.

# Run a Stateful Application Using MySQL

### Running Spring PetClinic on Kubernetes

The Spring PetClinic is a sample application designed to show how the Spring stack can be used to build simple, but powerful database-oriented applications.

The official version of PetClinic demonstrates the use of Spring Boot with Spring MVC and Spring Data JPA.

You can find more information in the link here

We are going to deploy the stack on Kubernetes which comprise of the following services:

Discovery Server:

- Config Server
- AngularJS frontend (API Gateway)
- Customers, Vets and Visits Services

The application uses MySQL as backend data store.

Pre-Requisites:

- Clone the repository that has the Kubernetes object resources files that compose our application:

```
git clone https://github.com/ahmedgabers/petclinic-kubernetes
cd petclinic-kubernetes
```

Installation

- Let's create an environment variable REPOSITORY_PREFIX that has the name of the repository which stores the Docker images for PetClinic services:<br>
  `export REPOSITORY_PREFIX=ahmedgabercod`
- Create the spring-petclinic namespace to contain all the resources we deploy for PetClinic:<br>
  `kubectl apply -f k8s/init-namespace/`
- The PetClinic microservices communicate together by networking provided by the Kubernetes services which are going to be used by our deployments. Run the following command to create the services in the spring-petclinic namespace:<br>
  `kubectl apply -f k8s/init-services`
- Our application services need a relational database in order to store customers, vets, and visits information. So we will use Helm to setup the MySQL backend for each service:
- - Add the bitnami repository and update Helm:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

- - Install the database instance for the vets service:<br>
    `helm install vets-db-mysql bitnami/mysql --namespace spring-petclinic --version 6.14.3 --set db.name=service_instance_db`
- - Install the database instance for the visits service:<br>
    `helm install visits-db-mysql bitnami/mysql --namespace spring-petclinic --version 6.14.3 --set db.name=service_instance_db`
- - Install the database instance for the customers service:<br>
    `helm install customers-db-mysql bitnami/mysql --namespace spring-petclinic --version 6.14.3 --set db.name=service_instance_db`

- Our deployment YAMLs have a placeholder called REPOSITORY_PREFIX so we'll be able to deploy the images from our repository by simply running a shell script:<br>
  `./scripts/deployToKubernetes.sh`
- Verify the pods are deployed:<br>
  `watch kubectl get pods -n spring-petclinic`

Once all Pods are in Running and Ready 1/1 state, visit the application from the tab in the left upper corner and explore the frontend:

From the 'Owners' dropdown list, select 'Register' and enter the information to register a new customer
Make sure the Telephone field contains no more than 10 numerical values without the '+' sign. e.g (7908645xxx)
From the 'Owners' dropdown list, select 'All' to verify the customer was added

- You can verify the data persisted in the MySQL database by running a temporary container with the mysql:5.7 image and running the mysql binary to execute the query:

```
# Export the MySQL password into an Environment Variable

export MYSQL_PASSWORD=$(kubectl get secret customers-db-mysql -n spring-petclinic -o jsonpath='{.data.mysql-root-password}' | base64 -d)

kubectl run mysql-client --image=mysql:5.7 --namespace spring-petclinic -i --rm --restart=Never --\
  mysql -h customers-db-mysql -uroot -p$MYSQL_PASSWORD<<EOF
USE service_instance_db;
SELECT * FROM owners;
EOF
```

You should be able to see the customer information that was added in the Database as well.

- Scaling deployments horizontally by adding multiple instances of the Application to tolerate surge in traffic is important, and we can easily achieve that in Kubernetes with a single command:<br>
  `kubectl scale deployments -n spring-petclinic {api-gateway,customers-service,vets-service,visits-service} --replicas=3`

- Verify the deployments are scaled:<br>
  `kubectl get pods -n spring-petclinic`

# Introduction to Kubestr

Kubestr is a collection of tools to discover, validate and evaluate your kubernetes storage options.

As adoption of Kubernetes grows so have the persistent storage offerings that are available to users. The introduction of CSI (Container Storage Interface) has enabled storage providers to develop drivers with ease. In fact there are around a 100 different CSI drivers available today. Along with the existing in-tree providers, these options can make choosing the right storage difficult.

Kubestr can assist in the following ways:

- Identify the various storage options present in a cluster.
- Validate if the storage options are configured correctly.
- Evaluate the storage using common benchmarking tools like FIO.

### Using Kubestr:

To install the tool

- Download the latest release:<br>
  `curl -sLO https://github.com/kastenhq/kubestr/releases/download/v0.4.17/kubestr-v0.4.17-linux-amd64.tar.gz`
- Unpack the tool and make it an executable chmod +x kubestr.

```
tar -xvzf kubestr-v0.4.17-linux-amd64.tar.gz -C /usr/local/bin
chmod +x /usr/local/bin/kubestr
```

To discover available storage options: Run `kubestr`

- To run an FIO test

Using kubestr we can test sequential read and write speeds which are expected to be high.

Random reads and writes can be tested as well. Moreover, random writes separates good drives from the bad ones.

- - Copy the test specification:

```
cat <<'EOF'>ssd-test.fio
[global]
bs=4k
ioengine=libaio
iodepth=1
size=1g
direct=1
runtime=10
directory=/
filename=ssd.test.file

[seq-read]
rw=read
stonewall

[rand-read]
rw=randread
stonewall

[seq-write]
rw=write
stonewall

[rand-write]
rw=randwrite
stonewall
EOF
```

- - Run the following command to start the test<br>
    `kubestr fio -f ssd-test.fio -s local-path`<br>
    Additional options like --size and --fiofile can be specified.
