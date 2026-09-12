# how to test a pvc's ability to bind + test local-path-provisioner
1. Create the test PVC and a pod that mounts it (the PVC alone won't bind — remember why: WaitForFirstConsumer waits for something to actually consume it before the provisioner acts):

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-test
  namespace: default
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: local-path-test
  namespace: default
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello-from-cp06 > /data/proof.txt && sleep 3600"]
      volumeMounts:
        - name: vol
          mountPath: /data
  volumes:
    - name: vol
      persistentVolumeClaim:
        claimName: local-path-test
EOF

2. Confirm the PVC bound and the pod landed where you expect:

kubectl get pvc local-path-test -o wide
kubectl get pod local-path-test -o wide   # check the NODE column — should be oci-w1

3. Confirm the PV itself is pinned to oci-w1:

kubectl get pv $(kubectl get pvc local-path-test -o jsonpath='{.spec.volumeName}') -o yaml | grep -A5 nodeAffinity

4. Confirm the data is actually on the /data volume, not the boot disk:

ssh oci-w1 'ls /data/local-path-provisioner/ && sudo find /data/local-path-provisioner -name proof.txt -exec cat {} \;'

5. Delete the pod and PVC, then prove Retain did its job:

kubectl delete pod local-path-test
kubectl delete pvc local-path-test
