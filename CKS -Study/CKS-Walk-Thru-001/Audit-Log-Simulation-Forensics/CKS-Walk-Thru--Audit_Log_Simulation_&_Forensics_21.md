[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)


## Audit Log Simulation & Forensics

---

📜 PART 2: Audit Log Simulation & Forensics
Now that your cluster is running, let’s simulate and review audit logs.

---

⚙️ Step 1: Enable Audit Logging (If Not Already)
Ensure kube-apiserver has this flag (check with ps aux | grep kube-apiserver):
ini



--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log


Use a basic policy like:



# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata


Then restart kube-apiserver (may require updating static pod manifest at /etc/kubernetes/manifests/kube-apiserver.yaml).

---
 
🔥 Step 2: Trigger Suspicious Events
For example:



kubectl auth can-i create clusterroles --as system:anonymous
kubectl get secrets --all-namespaces


Or apply a risky YAML like:
yaml



apiVersion: v1
kind: Pod
metadata:
  name: privileged
  namespace: dev
spec:
  containers:
  - name: shell
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      privileged: true

---


🔎 Step 3: Analyze Logs
View audit logs on the control plane node:
bash



sudo cat /var/log/kubernetes/audit.log | less


Search for:
• create events for pods, secrets

• userAgent fields

• as or impersonatedUser


You’re now practicing forensics!
Would you like to try simulating a malicious container action and watch Falco + audit logs react next?


---


audit logging still isn't working. Let's check the actual API server process to see if the audit flags are loaded:

bash
# Check the full command line of the running API server
sudo ps aux | grep kube-apiserver | grep -v grep

# Or check the process details
sudo cat /proc/$(pgrep -f "kube-apiserver")/cmdline | tr '\0' '\n' | grep audit
Also check the API server logs for audit-related errors:

bash
# Check API server logs for audit errors
sudo crictl logs $(sudo crictl ps --name kube-apiserver -q) 2>&1 | grep -i audit

# Or check kubelet logs
sudo journalctl -u kubelet --since "2 minutes ago" | grep -i audit
Most likely issue: The audit policy file path is wrong or not mounted. Check the volume mounts in your manifest:

bash
# Look for audit volume mounts in the running API server
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -A2 -B2 audit
From the manifest you showed earlier, I notice there are NO volume mounts for the audit policy or log directory. The API server needs these mount points to access the files.

Fix: Add volume mounts to the manifest
Edit the manifest:

bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
Add these volumeMounts under spec.containers[0].volumeMounts:

yaml
    volumeMounts:
    - mountPath: /etc/kubernetes/audit-policy.yaml
      name: audit-policy
      readOnly: true
      subPath: audit-policy.yaml
    - mountPath: /var/log/kubernetes
      name: audit-log
And add these volumes under spec.volumes:

yaml
  volumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log
Save and exit. Kubelet will auto-restart the API server. Then check again:

bash
# Wait 30 seconds
sleep 30

# Test with kubectl to generate logs
kubectl get pods -A

# Check audit log
sudo cat /var/log/kubernetes/audit.log | head -20


