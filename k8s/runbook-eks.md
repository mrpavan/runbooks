A comprehensive EKS Playbook / Runbook designed for an Operations Team. This runbook is structured to be easily scannable during a high-stress incident, moving from initial triage to specific problem scenarios. It is based on aws eks cluster enviornment, however steps defined here can be utilised for any k8s cluster troubleshooting or as a training guideline. 

---

# **EKS Cluster Operations Playbook (Runbook)**

**Document Owner:** [Your Team/Platform Team]  
**Last Updated:** [Date]  
**Purpose:** This document provides a standardized set of procedures for the Platform Operations Team to triage, investigate, and resolve common issues within our Amazon EKS clusters.

**When to use this Runbook:**
*   An alert has fired for an EKS cluster or a service running on it.
*   Users are reporting application failures, slowness, or unavailability.
*   During routine health checks of the cluster.

---

## **1. Pre-Requisites & Essential Tools**

Before you begin, ensure you have the following configured:

1.  **AWS CLI Access:** Your AWS CLI is configured with the correct IAM permissions to interact with EKS and EC2.
    ```bash
    # Verify your identity
    aws sts get-caller-identity
    ```
2.  **`kubectl` Access:** You have a valid `kubeconfig` file for the target cluster.
    ```bash
    # Update kubeconfig for the target cluster (replace placeholders)
    aws eks update-kubeconfig --region <aws-region> --name <cluster-name>

    # Verify you can connect to the cluster
    kubectl cluster-info
    kubectl get nodes
    ```
3.  **Essential CLI Tools:**
    *   `kubectl`: The Kubernetes command-line tool.
    *   `aws-cli`: The AWS command-line tool (v2 recommended).
    *   `k9s` (Highly Recommended): A terminal-based UI to manage Kubernetes clusters. Incredibly useful for fast navigation and troubleshooting.
    *   `helm`: If your applications are deployed via Helm.

---

## **2. Initial Triage: The First 5 Minutes**

When an issue is reported, follow these steps immediately to assess the "blast radius" and gather initial context.

### **Step 1: Assess Cluster Health at a High Level**

*   **Check Node Status:** Are all worker nodes `Ready`?
    ```bash
    kubectl get nodes -o wide
    # Look for any node with a status other than "Ready".
    ```
*   **Check for Failing Pods:** Find any pods across all namespaces that are not in a `Running` or `Succeeded` state.
    ```bash
    # This is your go-to command to find broken things quickly.
    kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
    ```
*   **Check Recent Cluster Events:** Events show what the Kubernetes scheduler and controllers have been doing. This is a goldmine for recent issues.
    ```bash
    # Get all events, sorted by the most recent first.
    kubectl get events --all-namespaces --sort-by='.lastTimestamp'
    ```

### **Step 2: Check Monitoring & Dashboards**

*   Open your primary observability dashboard (e.g., Grafana, Datadog, CloudWatch).
*   Look at key cluster-wide metrics:
    *   **CPU/Memory Utilization:** Are nodes or the cluster as a whole under pressure?
    *   **Pod Restarts:** Is there a spike in pod restarts for a specific deployment?
    *   **API Server Latency:** Is the Kubernetes control plane responding slowly?
    *   **Network I/O & Errors:** Are there unusual network patterns?

### **Step 3: Communicate**

*   Announce in your designated incident channel (e.g., Slack `#ops-incidents`) that you are investigating.
*   State the initial symptoms and what you've found so far (e.g., "Investigating high 5xx errors on Product-API. Seeing multiple pods in `CrashLoopBackOff` in the `product-api` namespace.").

---

## **3. Common Issue Playbooks**

Use the following playbooks based on the symptoms you've identified.

### **Playbook 3.1: Pod is `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff`**

**Symptom:** Application is down. `kubectl get pods` shows a pod is not `Running`.

#### **A. Pod is in `Pending` State**

A `Pending` pod means it cannot be scheduled onto a node.

1.  **Describe the Pod:** This is the most important step. Look at the "Events" section at the bottom of the output.
    ```bash
    kubectl describe pod <pod-name> -n <namespace>
    ```
2.  **Common Causes & Fixes:**
    *   **"Insufficient cpu/memory" (Unschedulable):**
        *   **Cause:** No node has enough available resources to meet the pod's `requests`.
        *   **Investigation:** Check node resources: `kubectl top nodes`. Check pod's requests: `kubectl get pod <pod-name> -o yaml`.
        *   **Fix:**
            *   Increase the size/number of nodes in the EKS Node Group (via ASG desired count or `eksctl scale nodegroup`).
            *   Lower the pod's resource `requests` (if appropriate).
    *   **"PodToleratesNoTaints" / "Failed to find a node that satisfies taints":**
        *   **Cause:** The pod doesn't have the necessary `tolerations` to be scheduled on a tainted node.
        *   **Fix:** Add the correct `toleration` to the pod's spec.
    *   **"PersistentVolumeClaim is not bound":**
        *   **Cause:** The PVC cannot find a matching Persistent Volume (PV) or the dynamic provisioner failed.
        *   **Investigation:** Check the status of the PVC: `kubectl describe pvc <pvc-name> -n <namespace>`. Check the logs of the storage provisioner (e.g., `ebs-csi-controller` in `kube-system`).

#### **B. Pod is in `CrashLoopBackOff`**

The container is starting, crashing, and being restarted by Kubernetes.

1.  **Check the Logs:** View the logs of the *current* crashing container.
    ```bash
    kubectl logs <pod-name> -n <namespace>
    ```
2.  **Check Previous Logs:** If the container crashes too quickly, you may need to see logs from the *previous* termination.
    ```bash
    kubectl logs --previous <pod-name> -n <namespace>
    ```
3.  **Describe the Pod:** Look at the "State" and "Last State" for exit codes and reasons.
    ```bash
    kubectl describe pod <pod-name> -n <namespace>
    ```
4.  **Common Causes & Fixes:**
    *   **Application Error:** The logs will contain an application-level stack trace or error message. This is often a bug that needs to be escalated to the development team.
    *   **Configuration Error:** The app is crashing because it's missing a required configuration file or environment variable. Check that referenced `ConfigMaps` and `Secrets` exist and are mounted correctly.
    *   **Out of Memory (OOMKilled):** The `describe pod` output will show `Reason: OOMKilled`. The container's memory `limit` is too low. Increase the memory limit in the deployment manifest.

#### **C. Pod is in `ImagePullBackOff` or `ErrImagePull`**

Kubernetes cannot pull the container image from the registry.

1.  **Describe the Pod:** The events will show the reason for the failure.
    ```bash
    kubectl describe pod <pod-name> -n <namespace>
    ```
2.  **Common Causes & Fixes:**
    *   **Typo in Image Name/Tag:** Verify the image name and tag in your deployment manifest are correct.
    *   **Image Does Not Exist:** The specified image tag doesn't exist in the container registry (e.g., ECR).
    *   **Authentication Failure:** The node does not have permission to pull from the registry.
        *   **For ECR:** Ensure the EKS Node IAM Role has permissions to pull from ECR (`ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`).
        *   **For other registries:** Ensure the `imagePullSecrets` are correctly configured and the secret contains a valid credential.

---

### **Playbook 3.2: Service is Unreachable (5xx Errors, Timeout)**

**Symptom:** Users cannot connect to a service, or connections are timing out.

1.  **Verify the Service and its Endpoints:**
    ```bash
    # Check if the Service object exists
    kubectl get svc <service-name> -n <namespace>

    # Describe the service. CRITICAL: Check the "Endpoints" field.
    kubectl describe svc <service-name> -n <namespace>
    ```
    *   **If `Endpoints` is `<none>`:** The service's `selector` does not match the labels on any running pods.
        *   **Fix:** Correct the `selector` in the Service manifest or the `labels` in the Pod/Deployment manifest so they match.
    *   **If `Endpoints` has IPs:** The service is correctly pointing to pods. The issue is likely with the pods themselves, networking, or the Ingress.

2.  **Check Network Policies:**
    *   A `NetworkPolicy` could be blocking traffic to the pods.
    ```bash
    # See if any policies are active in the namespace
    kubectl get networkpolicy -n <namespace>
    # Describe policies to understand their rules
    kubectl describe networkpolicy <policy-name> -n <namespace>
    ```

3.  **Check Ingress (if applicable):**
    ```bash
    # Get the ingress resource
    kubectl get ingress <ingress-name> -n <namespace>
    # Describe it to see rules and backend services
    kubectl describe ingress <ingress-name> -n <namespace>
    ```
    *   Check annotations for the Ingress Controller (e.g., AWS Load Balancer Controller).
    *   Check the logs of the Ingress Controller pods (e.g., `aws-load-balancer-controller` in `kube-system`).
    *   Check the AWS Load Balancer in the AWS Console for listener rules and target group health.

4.  **Test DNS Resolution from within the Cluster:**
    *   Sometimes the issue is CoreDNS.
    ```bash
    # Launch a temporary debug pod
    kubectl run dns-test --image=busybox:1.28 --rm -it -- nslookup <service-name>.<namespace>
    ```
    *   If `nslookup` fails, the problem is likely with CoreDNS. Check the `coredns` pod logs in the `kube-system` namespace.

---

### **Playbook 3.3: Node is in `NotReady` State**

**Symptom:** `kubectl get nodes` shows one or more nodes are not ready. Pods will be evicted from this node.

1.  **Describe the Node:** This gives you the status conditions and events related to the node.
    ```bash
    kubectl describe node <node-name>
    ```
    *   Look for conditions like `MemoryPressure`, `DiskPressure`, `PIDPressure`, or `KubeletReady: False`.

2.  **Check Kubelet on the Node:**
    *   The `kubelet` is the agent that connects the node to the control plane. If it's down, the node will be `NotReady`.
    *   SSH into the affected EC2 instance.
    ```bash
    # Check the status of the kubelet service
    sudo systemctl status kubelet

    # View the kubelet logs
    sudo journalctl -u kubelet -f
    ```

3.  **Check AWS EC2 Instance Status:**
    *   Go to the EC2 Console. Is the instance running? Did it fail a system or instance status check?

4.  **Check Networking:**
    *   A common EKS issue is running out of available IP addresses on the node, which prevents new pods from starting. This is related to the AWS VPC CNI.
    *   Check the `aws-node` pod logs on the affected node: `kubectl logs -n kube-system aws-node-<xxxxx>`

---

### **Playbook 3.4: Cluster Autoscaler Not Scaling Up/Down**

**Symptom:** Pods are `Pending` due to lack of resources, but new nodes are not being added.

1.  **Check Cluster Autoscaler Logs:**
    *   This is the most important source of information.
    ```bash
    # Find the cluster-autoscaler pod
    kubectl get pods -n kube-system | grep cluster-autoscaler
    # Tail its logs
    kubectl logs -f <cluster-autoscaler-pod-name> -n kube-system
    ```
    *   Look for messages about "unregistered" node groups, failed scaling activities, or pods that it cannot schedule.

2.  **Verify IAM Permissions:**
    *   The Cluster Autoscaler requires specific IAM permissions to modify Auto Scaling Groups (ASGs). Ensure the IAM role it uses has the correct policy attached.

3.  **Check ASG Settings in AWS Console:**
    *   Are the `Min` and `Max` sizes of the ASG configured correctly? Has the `Max` size been reached?
    *   Are the tags on the ASG correct (e.g., `k8s.io/cluster-autoscaler/enabled`)?

---

## **4. Post-Incident Actions**

1.  **Communicate Resolution:** Announce in the incident channel that the issue is resolved and what the fix was.
2.  **Create a Post-Mortem/RCA:** For significant incidents, schedule a blameless post-mortem to determine the root cause.
3.  **Update this Runbook:** If you discovered a new issue, a better command, or a clearer way to diagnose something, **update this document!** A runbook is a living document.
4.  **Create Follow-up Tickets:** Create tickets for any long-term fixes needed to prevent the issue from recurring.
