### Lab Guide: Create and Reconcile a Custom Resource Definition (CRD)

**Note: This lab is an extension of the Session 1 and will not be covered as part of the workshop**

This simplified lab guide will help you create a Custom Resource Definition (CRD), deploy a custom resource, and build a basic controller to reconcile the state of the custom resource.

---

### **Step 1: Set Up Your Environment**
Before starting, make sure your Kubernetes cluster is running and you have access to `kubectl`.

1. **Create a new namespace for the lab**  
   Create a namespace to keep everything organized:
   ```bash
   kubectl create namespace krm-crd-lab
   ```

---

### **Step 2: Create a Custom Resource Definition (CRD)**
We will define a new CRD called `CronTab` to manage scheduling configurations.

1. **Create a YAML file for the CRD**  
   Create a file named `crontab-crd.yaml` with the following content:
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

2. **Apply the CRD to the cluster**
   ```bash
   kubectl apply -f crontab-crd.yaml
   ```

3. **Verify the CRD is created**
   ```bash
   kubectl get crd
   ```

---

### **Step 3: Create a Custom Resource (CR)**
Now that the CRD is defined, we can create a custom resource that uses it.

1. **Create a YAML file for the CronTab resource**  
   Create a file named `my-crontab.yaml` with the following content:
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

2. **Apply the custom resource to the cluster**
   ```bash
   kubectl apply -f my-crontab.yaml
   ```

3. **Verify the resource is created**
   ```bash
   kubectl get crontabs -n krm-crd-lab
   ```

---

### **Step 4: Create a Basic Controller to Reconcile the Custom Resource**
We’ll create a simple controller that watches the `CronTab` resource and prints messages when it detects changes.

1. **Create a YAML file for the controller**  
   Create a file named `crontab-controller.yaml` with the following content:
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

2. **Apply the controller configuration**
   ```bash
   kubectl apply -f crontab-controller.yaml
   ```

3. **Run the controller in a pod (temporary setup)**  
   This command will run the controller inside a Python container:
   ```bash
   kubectl run crontab-controller --image=python:3.9 --restart=Never -n krm-crd-lab --command -- python /controller.py
   ```

4. **Check the controller logs to see reconciliation**  
   The controller will print messages as it reconciles the `CronTab` resources:
   ```bash
   kubectl logs crontab-controller -n krm-crd-lab
   ```

---

### **Step 5: Modify the Custom Resource**
Let’s change the number of replicas in the `CronTab` resource to see how the controller reacts.

1. **Update the `my-crontab.yaml` to change replicas**
   ```yaml
   replicas: 3
   ```

2. **Reapply the custom resource**
   ```bash
   kubectl apply -f my-crontab.yaml
   ```

3. **Check the logs again to observe reconciliation**
   ```bash
   kubectl logs crontab-controller -n krm-crd-lab
   ```

---

### **Step 6: Clean Up**
Once you're done with the lab, clean up the resources:

1. **Delete the custom resource**
   ```bash
   kubectl delete -f my-crontab.yaml
   ```

2. **Delete the CRD**
   ```bash
   kubectl delete -f crontab-crd.yaml
   ```

3. **Delete the namespace**
   ```bash
   kubectl delete namespace krm-crd-lab
   ```

---

### **Conclusion:**
In this lab, you learned how to create and manage a custom resource in Kubernetes using CRDs, and you built a basic controller to reconcile the state of the custom resource. This experience helps illustrate how Kubernetes can be extended beyond its built-in resources using CRDs.
