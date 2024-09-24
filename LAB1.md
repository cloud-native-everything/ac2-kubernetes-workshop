# Lab Guide: Experience KRM with Custom Resources and Controllers

This lab will demonstrate Kubernetes Resource Model (KRM) by creating a Custom Resource Definition (CRD) for a **CronJob**, applying changes to it, and using a basic controller to reconcile its state.

---

### **Step 0: Set Up Your Kubernetes Cluster (Kind)**
[kind](https://kind.sigs.k8s.io/) is a tool for running local Kubernetes clusters using Docker container “nodes”. kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

### Installation

```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Cluster Creation
To create a Kind based Kubernetes Cluster, issue the following command.
```bash
kind create cluster
# Allow the kind cluster to communicate with the later created containerlab topology
sudo iptables -I DOCKER-USER -o br-$(docker network inspect -f '{{ printf "%.12s" .ID }}' kind) -j ACCEPT
```

#### iptables command description

```
sudo iptables -I DOCKER-USER -o br-$(docker network inspect -f '{{ printf "%.12s" .ID }}' kind) -j ACCEPT
```

- `docker network inspect -f '{{ printf "%.12s" .ID }}' kind` - inspects the kind docker network, that the kind cluster is attached to. Extract from the json that is returned, the first 12 characters of the Id field.
- `sudo iptables -I DOCKER-USER -o br-$(...) -j ACCEPT` - as root insert a firewall rule to the DOCKER-USER chain, concerning the bridge with the name "br-<FIRST-12-CHAR-OF-THE-DOCKER-NETWORK-ID>" with the action ACCEPT.


## kubectl
`kubectl` is a command-line tool used to control and manage Kubernetes clusters. It allows developers and administrators to execute commands to create, monitor, and manage resources such as pods, services, deployments, and more within a Kubernetes cluster.

### Installation
To install `kubectl` issue the following command.
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```


### **Step 1: Set Up Your Environment**
1. **Access your Kubernetes cluster**  
   Ensure access to your Kubernetes cluster:
   ```bash
   kubectl get nodes
   ```
   Confirm the node is listed.

2. **Create a new namespace for the lab**  
   ```bash
   kubectl create namespace krm-crd-lab
   ```

---

### **Step 2: Create a Custom Resource Definition (CRD)**
1. **Define a CronTab CRD**  
   Create a YAML file `crontab-crd.yaml` with the following content to define the Custom Resource:
   ```yaml
   apiVersion: apiextensions.k8s.io/v1
   kind: CustomResourceDefinition
   metadata:
     name: crontabs.stable.example.com
   spec:
     group: stable.example.com
     names:
       kind: CronTab
       plural: crontabs
       singular: crontab
     scope: Namespaced
     versions:
       - name: v1
         served: true
         storage: true
         schema:
           openAPIV3Schema:
             type: object
             properties:
               spec:
                 type: object
                 properties:
                   cronSpec:
                     type: string
                   image:
                     type: string
                   replicas:
                     type: integer
   ```

2. **Apply the CRD**
   ```bash
   kubectl apply -f crontab-crd.yaml
   ```

3. **Verify the CRD**
   ```bash
   kubectl get crd
   ```

---

### **Step 3: Create a Custom Resource (CR)**
1. **Define a CronTab Resource**  
   Create a file called `my-crontab.yaml` with the following content to create a custom resource:
   ```yaml
   apiVersion: stable.example.com/v1
   kind: CronTab
   metadata:
     name: my-crontab
     namespace: krm-crd-lab
   spec:
     cronSpec: "* * * * */5"
     image: "busybox"
     replicas: 2
   ```

2. **Apply the Custom Resource**
   ```bash
   kubectl apply -f my-crontab.yaml
   ```

3. **Check the applied resource**
   ```bash
   kubectl get crontabs -n krm-crd-lab
   ```

---

### **Step 4: Create a Basic Controller to Reconcile the CR**
1. **Create a Controller using a ConfigMap**  
   Create a file `crontab-controller.yaml` for a simple controller to reconcile the state of the custom resource:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: crontab-controller
     namespace: krm-crd-lab
   data:
     controller.py: |
       import os
       import time
       import kubernetes
       from kubernetes import client, config

       config.load_incluster_config()
       api_instance = client.CustomObjectsApi()

       while True:
           crontabs = api_instance.list_namespaced_custom_object(
               group="stable.example.com", version="v1", namespace="krm-crd-lab", plural="crontabs"
           )
           for crontab in crontabs['items']:
               print(f"Reconciling CronTab: {crontab['metadata']['name']}")
           time.sleep(30)
   ```

2. **Apply the Controller**
   ```bash
   kubectl apply -f crontab-controller.yaml
   ```

3. **Run the Controller in a Pod (Temporary Setup)**
   ```bash
   kubectl run crontab-controller --image=python:3.9 --restart=Never -n krm-crd-lab --command -- python /controller.py
   ```

4. **Check Logs to See Reconciliation**
   ```bash
   kubectl logs crontab-controller -n krm-crd-lab
   ```

---

### **Step 5: Modify the CronTab Resource**
1. **Update the `replicas` field**
   Modify the `my-crontab.yaml` to change replicas:
   ```yaml
   replicas: 3
   ```

2. **Reapply the Custom Resource**
   ```bash
   kubectl apply -f my-crontab.yaml
   ```

3. **Observe Reconciliation in Controller Logs**
   ```bash
   kubectl logs crontab-controller -n krm-crd-lab
   ```

---

### **Step 6: Clean Up**
1. **Delete the Custom Resource**
   ```bash
   kubectl delete -f my-crontab.yaml
   ```

2. **Delete the CRD**
   ```bash
   kubectl delete -f crontab-crd.yaml
   ```

3. **Delete the Namespace**
   ```bash
   kubectl delete namespace krm-crd-lab
   ```

---

### Conclusion:
This lab demonstrates how KRM extends with Custom Resources (CRDs) and how a simple controller can reconcile their state. Students should now understand the flexibility of the Kubernetes API model beyond Pods and Deployments.
