This guide is created for my personal learning and testing purposes. It provides a comprehensive, step-by-step approach to building and deploying a Kubernetes Operator using Python and the Kopf framework

# Kubernetes Operator Tutorial

This tutorial guides you through the process of building a Kubernetes Operator using Python and the Kopf framework. The tutorial covers fundamental concepts like Custom Resource Definitions (CRDs), Custom Resources (CRs), and extends to deploying the operator on a Kubernetes cluster, testing it, and cleaning up resources.

## 1. Understanding the Fundamentals

### What is a Kubernetes Operator?

A Kubernetes Operator is a method of packaging, deploying, and managing a Kubernetes application. It uses Custom Resources (CRs) to extend the Kubernetes API and encapsulate application-specific logic in Custom Resource Definitions (CRDs). An Operator continuously monitors the state of the CRs and takes necessary actions to reconcile the actual state with the desired state defined in the CR.

### Key Concepts: CRD and CR

- **Custom Resource Definition (CRD):** A CRD defines new types of objects in Kubernetes, specifying the schema, structure, and validation rules for a Custom Resource (CR).
- **Custom Resource (CR):** A CR is an instance of the resource type defined by a CRD, representing the desired state of an application or component that the Operator will manage.

## 2. Setting Up the Development Environment

### Prerequisites

- Basic knowledge of Kubernetes and Python.
- A Kubernetes cluster (minikube, kind, or a managed service like AKS, GKE, etc.).
- kubectl installed and configured to interact with your cluster.

### Installing Required Tools

1. **Install Python**: Ensure Python is installed on your system by running:

   ```bash
   python3 --version
   ```

2. **Install Required Python Libraries**: Use pip to install the Kopf framework and the Kubernetes client library:

   ```bash
   pip install kopf kubernetes
   ```

## 3. Writing the Operator Logic in Python

### Creating the Operator Script

1. **Create a Python Script**: Start by creating a file named `my_operator.py`:

   ```python
   import kopf
   import kubernetes.client
   from kubernetes.client import AppsV1Api, V1Deployment, V1PodTemplateSpec, V1ObjectMeta, V1Container, V1PodSpec, V1LabelSelector

   # Load the Kubernetes configuration
   kubernetes.config.load_kube_config()

   @kopf.on.create('mycompany.com', 'v1', 'mywebapps')
   def create_fn(spec, **kwargs):
       name = kwargs['name']
       namespace = kwargs['namespace']

       replicas = spec.get('replicas', 1)
       image = spec.get('image', 'nginx:latest')
       
       deployment = V1Deployment(
           api_version="apps/v1",
           kind="Deployment",
           metadata=V1ObjectMeta(name=name, namespace=namespace),
           spec=kubernetes.client.V1DeploymentSpec(
               replicas=replicas,
               selector=V1LabelSelector(match_labels={"app": name}),
               template=V1PodTemplateSpec(
                   metadata=V1ObjectMeta(labels={"app": name}),
                   spec=V1PodSpec(containers=[V1Container(name=name, image=image)])
               ),
           ),
       )

       api = AppsV1Api()
       api.create_namespaced_deployment(namespace=namespace, body=deployment)
       kopf.info({'name': name}, reason='Created', message=f"Deployment {name} created with {replicas} replicas.")

   @kopf.on.delete('mycompany.com', 'v1', 'mywebapps')
   def delete_fn(spec, **kwargs):
       name = kwargs['name']
       namespace = kwargs['namespace']

       api = AppsV1Api()
       api.delete_namespaced_deployment(name=name, namespace=namespace)
       kopf.info({'name': name}, reason='Deleted', message=f"Deployment {name} deleted.")
   ```

### Explanation of the Code

- **Imports**: We import the necessary libraries like `kopf` for building the operator and `kubernetes.client` for interacting with Kubernetes resources.

- **Kopf Handlers**: The `@kopf.on.create` decorator is used to trigger the `create_fn` function whenever a new `MyWebApp` resource is created. Similarly, the `@kopf.on.delete` decorator handles resource deletions.

- **Deployment Logic**: The function creates a Kubernetes Deployment based on the specs defined in the `MyWebApp` resource.

## 4. Creating the Custom Resource Definition (CRD)

To let Kubernetes know about the new resource type `MyWebApp`, you need to create a CRD.

### CRD YAML File

Create a YAML file named `mywebapp-crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mywebapps.mycompany.com
spec:
  group: mycompany.com
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
                replicas:
                  type: integer
                image:
                  type: string
  scope: Namespaced #or scope: Cluster: Custom resources are accessible across the entire cluster, not tied to any
  names:
    plural: mywebapps
    singular: mywebapp
    kind: MyWebApp
    shortNames:
    - mwa
```

### Applying the CRD

Apply the CRD to your Kubernetes cluster:

```bash
kubectl apply -f mywebapp-crd.yaml
```

## 5. Deploying the Operator to the Cluster

### Option 1: Running Locally

If you're developing and testing locally, you can run the operator with:

```bash
python my_operator.py
```

### Option 2: Deploying as a Kubernetes Pod

1. **Create a Dockerfile**:

   ```dockerfile
   FROM python:3.8-slim

   WORKDIR /app

   COPY my_operator.py .
   COPY requirements.txt .

   RUN pip install -r requirements.txt

   CMD ["python", "my_operator.py"]
   ```

2. **Build and Push the Docker Image**:

   ```bash
   docker build -t my-operator:latest .
   docker tag my-operator:latest my-operator:latest
   docker push my-operator:latest
   ```

3. **Create a Kubernetes Deployment for the Operator**:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: my-operator
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: my-operator
     template:
       metadata:
         labels:
           app: my-operator
       spec:
         containers:
         - name: my-operator
           image: nginx:latest
   ```

   Apply the deployment:

   ```bash
   kubectl apply -f my-operator-deployment.yaml
   ```

## 6. Testing the Operator

### Creating a Custom Resource

Create a YAML file named `example-webapp.yaml`:

```yaml
apiVersion: mycompany.com/v1
kind: MyWebApp
metadata:
  name: example-webapp
  namespace: default
spec:
  replicas: 3
  image: nginx:latest
```

Apply the custom resource:

```bash
kubectl apply -f example-webapp.yaml
```

### Verify the Deployment

Check that the `nginx` deployment was created:

```bash
kubectl get deployments
kubectl get pods
```

### Delete the Custom Resource

To test the delete functionality, remove the `MyWebApp` resource:

```bash
kubectl delete -f example-webapp.yaml
```

Verify that the corresponding deployment is also deleted.

## 7. Cleaning Up

To delete all the resources created during this tutorial, follow these steps:

### Delete the Custom Resources

```bash
kubectl delete mywebapps.mycompany.com --all
```

### Delete the Operator Deployment

```bash
kubectl delete deployment my-operator
```

### Delete the CRD

```bash
kubectl delete crd mywebapps.mycompany.com
```

## Conclusion

This tutorial provided a comprehensive guide to building a Kubernetes Operator using Python and Kopf. By following the steps outlined, you should have a solid understanding of how Operators work, how to implement them, and how to manage their lifecycle within a Kubernetes cluster.
