# Lab 3: Backup Your Kubernetes Application

Many users from the virtualization world come to Kubernetes assuming that backup and data retention for applications is exactly the same as for legacy environments.

Nothing could be further from the truth! While Kubernetes provides high-availability parameters for containers, applications can only fully recover after a disaster scenario if the underlying data can also be secured and successfully restored, while being protected from corruption or loss.

All application and cluster control data must be successfully backed up.

As such, the Kubernetes platform is fundamentally different from all earlier compute infrastructures. It uses its own placement policy to distribute application components.

Containers can be dynamically rescheduled or scaled. New application components can be added or removed at any time.

A data management solution for Kubernetes needs to understand this cloud-native architectural pattern, be able to work with a lack of IP address stability, and deal with continuous change.

## Step by Step backup procedure

- Step 1: Add the Kasten K10 Helm repository
  The Helm package manager (v3.0.0+) and the Kasten Helm charts repository is required.

Installing K10 via helm will also automatically create a new Service Account to grant K10 the required access to Kubernetes resources.

- Step 2: Install K10
  Following a successful installation, there are several options for setting up access to the Kasten K10 dashboard.

- Step 3: Backup Policy Creation
  K10 gives you the ability, with extremely minor application modifications, to add functionality to backup, restore, and migrate application data in an efficient and transparent manner.

- Step 4: Restore from Backup
  Restore the data using K10 by selecting the appropriate restore point.

Today’s complex enterprise IT workloads require the scale and flexibility that only Kubernetes can deliver. But security issues facing Kubernetes users continue to disrupt critical operations, particularly as ransomware inflicts increasing damage on enterprises.

Conservative estimates put global ransomware losses upwards of $20 billion in 2020, with 2021 expected to be even worse.

While the primary goal is to be able to prevent a ransomware attack altogether, it is equally as important to plan for how to recover from one. Hardened backup capabilities are a must and enterprise IT teams should be aware of the different operating requirements for Kubernetes applications in contrast to legacy, monolithic systems.

**Application State and Configuration Data** – Each application must include the state that spans across storage volumes and databases (NoSQL/relational), as well as configuration data included in Kubernetes objects such as configmaps and secrets.

**Snapshots versus Backups** – Snapshots are typically stored alongside primary data and are not fully isolated. This means that, alone, snapshots might not be available if something happens to the cluster holding them, rendering them unreliable for recovery and long-term data retention due to potential data loss.

**Application Security for Cloud-Native Environments** – Cloud-native environments have shifted the importance and function of application security. End-to-end encryption and customer-owned management capabilities are paramount and should include integrated authentication and role-based access control (RBAC).

Lastly, application security must allow for a quick recovery from ransomware attacks.

**Prioritizing Recovery**

Despite planning for disaster, should a ransomware attack succeed, recovery is an organization’s next line of defense. Ransomware attacks are not one-size-fits all and attackers work diligently to find the right targets, as well.

In the case of Kubernetes, an attack on a cluster may stem from something as “simple” as an overlooked, unauthenticated endpoint or an unpatched vulnerability. In the event of a successful attack, fast restores are essential to protecting sensitive data from being exploited and resuming business operations quickly.

Enterprise IT teams run and maintain thousands of applications across different locations using platforms like Kubernetes that enable automation – overseeing all of them manually is a task nearly beyond human capabilities.

Tools for backup and recovery functions should also promote automation and integrate seamlessly into existing workflows. Enabling immutability, creating backups with unique code paths, protecting backups for maximum effectiveness and enabling seamless restores are part of a robust ransomware data protection strategy.

If an organization is attacked by ransomware, and there is no plan to defend against it, the attack can result in significant business disruption and major financial losses, even without paying ransom (which a business should never do).

A robust ransomware data protection strategy can enable organizations to recover from data loss and ransomware attacks with sustained success.

Beyond tooling, IT teams need to educate stakeholders on how to avoid ransomware and detect phishing campaigns, suspicious websites and other scams, like social engineering, while security resources should be aimed at hardening application infrastructure – systems and networks – and maintaining all software updates on a regular basis.

For applications running on Kubernetes, IT teams need to be aware of their unique requirements to protect them and ensure that backup and recovery capabilities are hardened accordingly. Kubernetes is the unifying fabric of modern computing. Using it to mitigate the most pressing data threats in today’s risk landscape is one of the best defenses yet.

# Getting Started

## **Add the Kasten Helm 10 repository**

- Let's start setting our environment up by adding the the Kasten K10 Helm repository to the system:<br>
  `helm repo add kasten https://charts.kasten.io/`

## **Install MySQL and Create a Demo Database**

To experiment with backup and recovery of a cloud-native application, we will install MySQL and create a database in this step.

- First, install MySQL using the following commands:<br>

```
kubectl create namespace mysql
helm install mysql bitnami/mysql --namespace=mysql
```

- To ensure that MySQL is running, check the pod status to make sure they are all in the Running and Ready 1/1 state:<br>
  `watch -n 2 "kubectl -n mysql get pods"`

- Once all pods have a Running and Ready 1/1 status, hit CTRL + C to exit watch and then run the following commands to create a local database.<br>

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "CREATE DATABASE k10demo"
```

## Install K10 and Configure Storage

**Install Kasten K10**

- In this step, we will actually install K10 by running the following commands:<br>

`helm install k10 kasten/k10 --namespace=kasten-io --create-namespace`

- To ensure that Kasten K10 is running, check the pod status to make sure they are all in the Running and Ready 1/1, 2/2 state:<br>
  `watch -n 2 "kubectl -n kasten-io get pods"`

Once all pods have a Running and Ready 1/1, 2/2 status, hit CTRL + C to exit watch.

- Configure the Local Storage System
- - Once K10 is running, use the following commands to configure the local storage system.<br>
    `kubectl annotate volumesnapshotclass csi-hostpath-snapclass k10.kasten.io/is-snapshot-class=true`

## View K10 Dashboard

**Expose the K10 dashboard**

- While not recommended for production environments, let's set up access to the K10 dashboard by creating a NodePort. Let's first create the configuration file for this:<br>

```
cat > k10-nodeport-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: gateway-nodeport
  namespace: kasten-io
spec:
  selector:
    service: gateway
  ports:
  - name: http
    port: 8000
    nodePort: 32000
  type: NodePort
EOF
```

- Now, let's create the actual NodePort Service<br>
  `kubectl apply -f k10-nodeport-svc.yaml`

**View the K10 Dashboard**
Once completed, you should be able to view the K10 dashboard in the other tab on the left. We recommend taking the K10 dashboard tour at this point.

## Backup Policy Creation

We will now create a policy to backup MySQL to the previously configured object storage location.

From the main K10 dashbboard, click on the Policies card. There should be no policies visible at this point.

Click Create New Policy and:<br>

- Give the policy the name: backup-mysql
- Select Snapshot for the Action
- Select Hourly for the Action Frequency
- Leave the Snapshot Retention selection as-is
- Select By Name for Select Applications and then, from the dropdown, select mysql
- Leave all other settings as-is and select Create Policy

**Running the Policy**

The above policies will only run at the scheduled time (by default, at the top of the hour). To run the policies manually for the first time, click on Run Once on the Policies page. Confirm by clicking Run Policy and then go back to the main dashboard to view the job in action. Verify that the job has successfully completed.

## Restore from Backup

Now that we have a MySQL backup, let's go simulate accidental data loss and then recover the system from that loss.

**Causing Data Loss**

- For the purposes of this test drive, we will simply drop the k10demo database we had created earlier.<br>

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "DROP DATABASE k10demo"
```

- Verify that the database has been deleted by running:<br>
  `kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SHOW DATABASES LIKE 'k10demo'"`

**Recovering Data**

To recover the databases, go to the K10 dashboard, click Applications, and then select Restore on the MySQL card.

Click on a recent restore point and then select the Exported restore point (this is stored in the object storage system instead of as a non-durable snapshot on the storage system). In this case, we will select the default Application Name option to restore in place (Restore as "mysql"). Leave all other selections as-is, click on Restore, and confirm the action.

Return to the main dashboard to view the Restore job and verify that it completes successfully.

- To verify that our data was recovered, run the following command in the terminal to view the restored database:<br>

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
kubectl exec -it --namespace=mysql $(kubectl --namespace=mysql get pods -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SHOW DATABASES LIKE 'k10demo'"
```
