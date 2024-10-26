
# Moving StatefulSet Pods Between Availability Zones in AWS EKS

Recently, in an AWS EKS cluster, I needed to move StatefulSet pods from one availability zone (`us-east-1b`) to another (`us-east-1a`) due to certain requirements. Each pod had a Persistent Volume Claim (PVC) attached to it using EBS volumes.

Here are the challenges I faced and how I solved them.

## What is a StatefulSet in Kubernetes?
A StatefulSet in Kubernetes is designed to manage stateful applications, ensuring that each pod has a unique identity and maintains a persistent storage volume. This setup allows data persistence across pod restarts, which is essential for applications like databases.

## Challenges Faced
- **Maintaining High Availability:** I couldn't remove the node as it was a production environment and needed to maintain high availability.
- **Cross-AZ Volume Attachment:** When trying to move the pod to a node in a different availability zone (`us-east-1a`), the volume from `us-east-1b` couldn't be attached to an instance/node in `us-east-1a` due to network and data locality constraints.

## How I Solved the Issue

### 1. Cordoning Nodes in `us-east-1b` 🚧
I cordoned all the nodes running in `us-east-1b`, including the one where the ReplicaSet pod I wanted to move was running. This ensured the pod would not be rescheduled to those nodes.

```bash
kubectl cordon <node-name>
```
**Example:**
```bash
kubectl cordon ip-10-4-158-210.ec2.internal
kubectl cordon ip-10-4-158-211.ec2.internal
kubectl cordon ip-10-4-158-212.ec2.internal
```

### 2. Draining the Node Running the ReplicaSet Pod 💧
Next, I drained the node that was running the ReplicaSet pod. The `kubectl drain` command safely removes all non-DaemonSet and non-mirrored pods from a node, preparing it for maintenance.

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
**Example:**
```bash
kubectl drain ip-10-4-158-211.ec2.internal --ignore-daemonsets --delete-emptydir-data
```

### 3. PVC Cleanup 🗑️
#### What is PVC?
A PersistentVolumeClaim (PVC) in Kubernetes is a request for storage by a pod, allowing it to use a PersistentVolume (PV) that provides long-term, stable storage.

After deleting the node (e.g., `ip-10-4-158-211.ec2.internal`), the PVC gets freed up. The pod can then be scheduled to the new node (`us-east-1a`), but it may not start. You can check the reason using:

```bash
kubectl describe pod <pod_name>
```

If you try to remove the PVC normally, it might get stuck in a terminating state. Here's how to forcefully remove the PVC (make sure to back up important data if there's no auto-sync mechanism):

```bash
kubectl patch pvc <pvc_name> -p '{"metadata":{"finalizers":null}}'
kubectl patch pv <pv_id> -p '{"metadata":{"finalizers":null}}'
kubectl delete pvc <pvc_name> --grace-period=0 --force
kubectl delete pv <pv_id>
kubectl delete pod <pod_name>
```

After this, the pod will be recreated, a new PVC will be created and attached to the pod properly, and the pod will be in a running state. ✅
