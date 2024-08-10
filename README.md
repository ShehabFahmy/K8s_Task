Create a CronJob that runs a Python script every 3 minutes. The script is stored in a ConfigMap and modifies a file in a Persistent Volume (PV) attached to the pod as a Persistent Volume Claim (PVC) with hostPath. After modifying the script, you will check the changes in the file stored in the PV.
## Shell Script
- `pv.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-task
spec:
  storageClassName: "stclass-task"
  capacity:
    storage: 20Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/my-pv"
```
- `pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-task
spec:
  storageClassName: "stclass-task"
  volumeName: pv-task
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Mi
```
- `cm.yaml`:
```sh
kubectl create cm cm-task --from-file=script.sh
```
or
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-task
data:
  script.sh: |
    #!/bin/bash
    echo "$(date) |  Hello" >> /home/my-storage/output.txt
```
- `temporary-pod.yaml`: just a temporary Pod to test spec.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-task
spec:
  volumes:
  - name: local-storage
    persistentVolumeClaim:
      claimName: pvc-task
  - name: cm-volume
    configMap:
      name: cm-task
  containers:
  - name: ubuntu
    image: ubuntu
    imagePullPolicy: IfNotPresent
    command: ["/bin/bash", "-c"]
    args:
    - |
      # Copy the file out of the ConfigMap Volume as it is Read-only
      # cp /home/my-storage/scripts/script.sh /home/my-storage/
      # Make the script executable
      # chmod +x /home/my-storage/script.sh
      # Run the script
      bash /home/my-storage/scripts/script.sh
      # Keep the container running for an hour
      # sleep 3600
    volumeMounts:
    - name: local-storage
      mountPath: /home/my-storage      
    - name: cm-volume
      mountPath: /home/my-storage/scripts
  restartPolicy: OnFailure
```
- `cronjob.yaml`: where we we put the `temporary-pod.yaml`'s spec.
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-job-task
spec:
  schedule: "*/3 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          volumes:
          - name: local-storage
            persistentVolumeClaim:
              claimName: pvc-task
          - name: cm-volume
            configMap:
              name: cm-task
          containers:
          - name: ubuntu
            image: ubuntu
            imagePullPolicy: IfNotPresent
            command: ["/bin/bash", "-c"]
            args:
            - |
              # Copy the file out of the ConfigMap Volume as it is Read-only
              # cp /home/my-storage/scripts/script.sh /home/my-storage/
              # Make the script executable
              # chmod +x /home/my-storage/script.sh
              # Run the script
              bash /home/my-storage/scripts/script.sh
              # Keep the container running for an hour
              # sleep 3600
            volumeMounts:
            - name: local-storage
              mountPath: /home/my-storage      
            - name: cm-volume
              mountPath: /home/my-storage/scripts
          restartPolicy: OnFailure
```
## Python Script
- Same PV and PVC
- `script.py`:
```python
with open("/home/my-storage/output.txt", "w") as f:
    f.write("Hello from the Python script!")
```
- Image used will be `python` instead of `ubuntu`
- The command will be `python /home/my-storage/scripts/script.py`
