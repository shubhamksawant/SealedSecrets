### Secure Secrets Management Using Sealed Secrets in Kubernetes

Managing sensitive data securely in Kubernetes is a challenge due to the limitations of traditional Kubernetes Secrets, which store data encoded in base64 but not encrypted. Bitnami's **Sealed Secrets** provides a robust solution to this problem by allowing encryption of secrets outside the cluster and ensuring they are only decrypted by the intended Kubernetes cluster.

---

#### **Problem Statement**

Managing sensitive data like passwords, OAuth tokens, and SSH keys in Kubernetes is a critical challenge. By default, Kubernetes Secrets are encoded in base64 and stored in etcd, which poses significant risks:

- **Etcd Vulnerability**: Base64 encoding is not encryption. If etcd is compromised or unauthorized access is gained, secrets can be decoded easily.
- **Source Control Risks**: Storing Secrets in version control can lead to accidental exposure and compromise sensitive information.

DevOps and SRE teams often struggle to balance security with the operational need to manage secrets as part of their application's deployment lifecycle.

---

#### **Solution: Sealed Secrets**

Sealed Secrets provide a secure and user-friendly approach to secret management in Kubernetes. They allow sensitive data to be encrypted and safely stored in source control, ensuring secure handling throughout the CI/CD pipeline.

---

#### **How Sealed Secrets Work**

1. **Controller Deployment**: A Sealed Secrets Controller is installed in the Kubernetes cluster. This controller manages the encryption and decryption process using asymmetric cryptography.
   
2. **Encrypting Secrets**:
   - Users encrypt a Kubernetes Secret using the `kubeseal` CLI tool and the controller's **public key**.
   - This generates a `SealedSecret` YAML file, which is safe to commit to source control, even in public repositories.

3. **Applying Sealed Secrets**:
   - The SealedSecret is applied to the Kubernetes cluster like any other resource.
   - The Sealed Secrets Controller decrypts it using its **private key** and creates a standard Kubernetes Secret.

4. **Using Secrets**:
   - Pods access the standard Secret just as they would with a traditional Kubernetes Secret.
   - The sensitive data remains encrypted during transit and in storage outside the cluster.

---

#### **Key Benefits**

- **End-to-End Security**: Sensitive data is encrypted from creation to usage, ensuring it is never exposed in plaintext outside the cluster.
- **Source Control Integration**: Enables secure versioning of secrets alongside application manifests.
- **Granular Access Control**: Secrets are only decrypted within the target cluster, ensuring their confidentiality even if the SealedSecret is publicly accessible.
- **Cluster-Specific Encryption**: The encrypted Secret is bound to the target cluster's Sealed Secrets Controller, preventing reuse in other clusters.
- **Encryption with Asymmetric Cryptography**: Uses a public-private key pair. Secrets are encrypted using the controller's public key and can only be decrypted by the private key held by the controller in the cluster.
**CI/CD Pipeline Integration**: Ensures secrets remain encrypted throughout the CI/CD process, maintaining security until deployment.
**Namespace and Name Specificity**: SealedSecrets are bound to a namespace and name, providing granular access control.


---

#### **Use Cases**

1. **CI/CD Pipelines**:
   - Securely store and manage secrets used in deployment pipelines.
   - Avoid plaintext secrets in pipeline logs or configuration files.

2. **Multi-Environment Deployments**:
   - Commit environment-specific SealedSecrets to a shared repository.
   - Ensure secrets are accessible only within their respective clusters.

3. **Compliance and Auditing**:
   - Demonstrate robust secret management for regulatory compliance.
   - Maintain an audit trail of encrypted secrets in version control.

---

### Step-by-Step Guide to Using Sealed Secrets:

#### **1. Install Sealed Secrets Controller**
Deploy the Sealed Secrets controller in your Kubernetes cluster:
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/controller.yaml
```
This deploys the controller in the `kube-system` namespace. Verify installation:
```bash
kubectl get pods -n kube-system | grep sealed-secrets
```

#### **2. Install `kubeseal` CLI**
Download and install the CLI:
```bash
curl -LO https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/kubeseal-0.26.0-linux-amd64.tar.gz
tar -xzvf kubeseal-0.26.0-linux-amd64.tar.gz
mv kubeseal /usr/bin/
kubeseal --version
```

#### 3. How to Use Sealed Secrets in Kubernetes

Sealed Secrets provide a secure and efficient way to manage secrets in Kubernetes by encrypting them before storing in a repository. This guide outlines the steps to use Sealed Secrets in your Kubernetes environment.

---

### **Prerequisites**

- A running Kubernetes cluster.
- `kubectl` installed and configured.
- The Sealed Secrets controller installed in the cluster.
- The `kubeseal` CLI tool installed on your local machine.

---
### **Step 1: Create a Kubernetes Secret**

Create a standard Kubernetes Secret manifest (`secret.yaml`) as follows:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
data:
  username: bXl1c2VybmFtZQ==  # base64 encoded value
  password: bXlwYXNzd29yZA==  # base64 encoded value
```
---
### **Step 2: Encrypt the Secret Using `kubeseal`**

Use the `kubeseal` CLI to encrypt the Secret manifest. The output will be a SealedSecret that is safe to store in version control.

1. Encrypt the Secret:
   ```bash
   kubeseal --format yaml < secret.yaml > sealed-secret.yaml
   ```

   - The `kubeseal` command uses the public key of the Sealed Secrets controller to encrypt the Secret.
   - The `--format` option specifies the output format (YAML or JSON).

2. The output `sealed-secret.yaml` will look like this:
   ```yaml
   apiVersion: bitnami.com/v1alpha1
   kind: SealedSecret
   metadata:
     name: my-secret
     namespace: default
   spec:
     encryptedData:
       username: Agd2z+fEdz...
       password: Tm90RZXJq...
   ```

### **Step 3: Apply the SealedSecret to the Cluster**

Commit the `sealed-secret.yaml` file to your repository if needed, then apply it to the cluster:
```bash
kubectl apply -f sealed-secret.yaml
```

The Sealed Secrets controller will detect the SealedSecret, decrypt it using its private key, and create a standard Kubernetes Secret in the specified namespace.

---

### **Step 4: Verify the Decrypted Secret**

Check that the Secret was created successfully:
```bash
kubectl get secret my-secret -o yaml
```

The sensitive data will be base64 encoded, as per Kubernetes standards.

---

### **Step 5: Use the Secret in Your Application**

Update your Pod or Deployment manifests to use the Secret. For example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: my-container
    image: nginx
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

---

### **Additional Notes**

- The SealedSecret is cluster-specific and namespace-specific.
- To encrypt a Secret for a different namespace, specify the `--namespace` option in the `kubeseal` command.
- To fetch the controller's public key manually (if needed):
  ```bash
  kubeseal --fetch-cert > pub-cert.pem
  ```
---

### Example Deployment
Here’s how Sealed Secrets integrate into the existing MySQL and Adminer setup:

#### **Original Secret Definition (Insecure)**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: "cGFzc3dvcmQxMjM="
  MYSQL_DATABASE: "bXlkYg=="
  MYSQL_USER: "ZGItdXNlcg=="
  MYSQL_PASSWORD: "cGFzc3dvcmQxMjM="
```

#### **SealedSecret Definition (Secure)**
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysql-secret
  namespace: default
spec:
  encryptedData:
    MYSQL_ROOT_PASSWORD: "AgCZy85yQ..."
    MYSQL_DATABASE: "AgBA99ARj..."
    MYSQL_USER: "AgA1il6d8..."
    MYSQL_PASSWORD: "AgCZy85yQ..."
```

#### **Applying the SealedSecret**
```bash
kubectl apply -f mysql-sealedsecret.yaml
```

The cluster decrypts this SealedSecret and creates the required Secret, enabling your MySQL and Adminer services to use the credentials securely.

---

### Benefits in Practice
1. **Reduced Risk of Exposure**:
   Even if the YAML file is exposed, the sensitive data remains encrypted.

2. **Version Control Safety**:
   The SealedSecret file can be safely committed to Git without compromising security.

3. **Cluster-Specific Decryption**:
   Secrets are only usable in the intended cluster, enhancing security in multi-cluster environments.
