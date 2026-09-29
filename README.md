# Injecting Secret as Environment Variables in Kubernetes

## Overview

Kubernetes `Secret` is used to store and manage sensitive information such as passwords, OAuth tokens, and SSH keys.

A Secret can be consumed by a Pod in different ways. In this lab, the Secret is created first and then specific Secret keys are injected into a container as environment variables.

The lab demonstrates:

* Creating a Kubernetes Secret
* Creating a Secret using literal values
* Creating a Secret using a YAML manifest
* Encoding values using Base64
* Referencing Secret keys from a Pod
* Injecting Secret values as environment variables
* Verifying the injected variables from the running Pod

---

## 1. Lab Objective

Create a Secret named `my-secret` and use it inside a Pod named `my-app-pod`.

The Pod will use the `alpine` image.

The Secret contains the following values:

| Key        | Value         |
| ---------- | ------------- |
| `username` | `my-app-user` |
| `password` | `s3cr3t`      |

The overall flow is:

```text
Secret
  |
  | secretKeyRef
  v
Pod
  |
  v
Container Environment Variables
```

---

## 2. How Kubernetes Secret Injection Works

Secrets are stored in Kubernetes' central datastore, `etcd`. A Secret can then be consumed by multiple Pods when their specifications reference it.

For this lab, the flow is:

```text
Create Secret
      |
      v
  my-secret
   /      \
username  password
   |         |
   +----+----+
        |
        v
    secretKeyRef
        |
        v
   my-app-pod
        |
        v
USERNAME / PASSWORD
```

The Pod does not directly define the actual Secret values. Instead, it references the required keys from `my-secret`.

---

# 3. Create the Secret

There are multiple ways to create a Kubernetes Secret.

In this lab, two approaches are demonstrated:

1. Using literal values with `kubectl`
2. Using a YAML manifest

---

## 3.1 Create Secret Using Literal Values

The following command creates a Secret named `my-secret` with two keys:

```bash
kubectl create secret generic my-secret \
  --from-literal=username=my-app-user \
  --from-literal=password=s3cr3t
```

This creates:

```text
Secret: my-secret

username = my-app-user
password = s3cr3t
```

The command-based approach is useful when creating a Secret quickly from the command line.

---

# 4. Create Secret Using YAML

A Secret can also be defined declaratively using a YAML manifest.

For the `data` field, the values are Base64 encoded first.

## 4.1 Encode the Values

Encode the username:

```bash
echo -n 'my-app-user' | base64
```

Output:

```text
bXktYXBwLXVzZXI=
```

Encode the password:

```bash
echo -n 's3cr3t' | base64
```

Output:

```text
czNjcjN0
```

These encoded values are then used in the Secret manifest.

---

## 4.2 Secret Manifest

Create `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: bXktYXBwLXVzZXI=
  password: czNjcjN0
```

Apply the manifest:

```bash
kubectl apply -f secret.yaml
```

The source lab uses an `Opaque` Secret containing the two encoded data keys.

---

# 5. Define a Pod That Uses the Secret

Now create a Pod that consumes the Secret.

Create:

```text
my-app-pod.yaml
```

with the following definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod

spec:
  containers:
  - name: my-app-container
    image: alpine
    command: ["/bin/sh"]
    args: ["-c", "env && sleep 3600"]

    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username

    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
```

The important part is the `env` section.

---

## 5.1 Understanding `secretKeyRef`

The first environment variable:

```yaml
- name: USERNAME
  valueFrom:
    secretKeyRef:
      name: my-secret
      key: username
```

means:

```text
Environment Variable
USERNAME
     |
     v
Secret: my-secret
     |
     v
Key: username
     |
     v
my-app-user
```

Similarly:

```yaml
- name: PASSWORD
  valueFrom:
    secretKeyRef:
      name: my-secret
      key: password
```

maps the `password` key from `my-secret` to the `PASSWORD` environment variable.

So inside the container, the application can access:

```text
USERNAME
PASSWORD
```

without defining the original Secret values directly in the Pod manifest.

---

# 6. Deploy the Pod

Apply the Pod definition:

```bash
kubectl apply -f my-app-pod.yaml
```

The source lab uses this command to create the Pod after defining the Secret references.

---

# 7. Verify the Pod

Check whether the Pod was created successfully:

```bash
kubectl get pod
```

Expected output should show:

```text
NAME          READY   STATUS    RESTARTS   AGE
my-app-pod    1/1     Running   0          ...
```

The exact output will depend on the Kubernetes cluster environment.

---

# 8. Verify the Environment Variables

The Pod definition runs:

```bash
env && sleep 3600
```

inside the Alpine container.

Therefore, the environment variables can be verified from the Pod logs:

```bash
kubectl logs my-app-pod
```

The output should contain entries similar to:

```text
USERNAME=my-app-user
PASSWORD=s3cr3t
```

The lab uses Pod logs to verify that the Secret values were successfully injected as environment variables.

---

# 9. Complete Lab Flow

For a quick reference, the complete workflow is:

### Step 1 — Create Secret

```bash
kubectl create secret generic my-secret \
  --from-literal=username=my-app-user \
  --from-literal=password=s3cr3t
```

### Step 2 — Create Pod

```bash
kubectl apply -f my-app-pod.yaml
```

### Step 3 — Check Pod

```bash
kubectl get pod
```

### Step 4 — Verify Secret Injection

```bash
kubectl logs my-app-pod
```

---

# 10. Key Kubernetes Concepts

### Secret

A Kubernetes object used for managing sensitive configuration data.

### `secretKeyRef`

Used when a specific key from a Secret needs to be mapped to an environment variable.

### `env`

Defines environment variables inside the container.

### `Opaque`

The Secret type used in this lab.

### Base64

The YAML example uses Base64-encoded values under the Secret `data` field. Base64 encoding should not be confused with encryption.

---

# 11. Operational Takeaway

The important part of this exercise is not simply creating a Secret.

The more useful concept is the separation between:

```text
Application
     |
     +---- Container Image
     |
     +---- Pod Definition
     |
     +---- Sensitive Configuration
                    |
                    v
                  Secret
```

The application workload can reference the required Secret keys instead of defining those values directly inside the container configuration.

This pattern becomes particularly useful when the same application needs to run across different environments while the sensitive configuration changes independently.

---

# 12. Lab Files

A simple repository structure for this exercise can be:

```text
kubernetes-secret-env/
├── README.md
├── secret.yaml
└── my-app-pod.yaml
```

Where:

```text
secret.yaml
    |
    +-- Creates my-secret

my-app-pod.yaml
    |
    +-- Creates my-app-pod
    +-- References my-secret
```

---

## Conclusion

This lab demonstrates a basic but important Kubernetes pattern: creating a Secret and injecting selected Secret keys into a Pod as environment variables.

The core relationship is:

```text
Secret
  ↓
secretKeyRef
  ↓
Environment Variable
  ↓
Container
  ↓
Application
```

Once this flow is clear, it becomes much easier to understand how Kubernetes workloads consume sensitive configuration without placing the original values directly in the application container definition.
