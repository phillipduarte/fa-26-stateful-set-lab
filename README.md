# Deployment vs StatefulSet

First double check that your cluster has a storage class. If this does not show anything, then please ask on Ed for help.

```bash
kubectl get storageclass
```

## 1. Deployment

```bash
kubectl apply -f deployment.yml
kubectl get pods -l app=note
kubectl get pvc
```

The pods stay `Pending` for a few seconds while the volume is created. Run the `get pods` command again until both rows say `Running` before you answer, and before you write a note.

**Q1.** What do you observe about the pods?

get pods returns the 2 pods with the same prefix note but then a distinct suffix. They are both ready and running. The pods share
a volume that we can see with the get pvc.

note-65886f65df-f7jmx
note-65886f65df-qwksh


## 2. Write on one pod, read on the other

Copy the two pod names from the previous command. Run these one at a time, with those names filled in:

```bash
kubectl exec POD_A -- sh -c 'echo hello-from-a > /data/note.txt'
kubectl exec POD_B -- cat /data/note.txt
kubectl get pvc
```

**Q2.** What do you observe about the note, and about the volumes?

The note got written to by the first pod and then it got read by the second pod
to get "hello-from-a." They share the same volume as we see when we run get pv.
It's also persistent.

## 3. Remove the Deployment

```bash
kubectl delete -f deployment.yml
kubectl get pods -l app=note
kubectl get pvc
```

Wait until the pods and the `data-note` claim are gone before continuing.

## 4. StatefulSet

Run these two commands back to back. `-w` keeps printing pod updates until you press Ctrl-C. If you wait too long to start it, both pods will already be `Running` and you will miss the order.

```bash
kubectl apply -f stateful.yml
kubectl get pods -l app=note -w
```

**Q3.** In what order did the pods start, and what are they named?

The note-0 pod started first and then the note-1 pod started second.

## 5. A different note on each pod

```bash
kubectl exec note-0 -- sh -c 'echo hello-from-0 > /data/note.txt'
kubectl exec note-1 -- sh -c 'echo hello-from-1 > /data/note.txt'
kubectl exec note-0 -- cat /data/note.txt
kubectl exec note-1 -- cat /data/note.txt
kubectl get pvc
```

**Q4.** How does this differ from what you saw in Q2?

This differs as the output just returns the message written by each note individually instead
of them both being in the volume. This implies they are different volumes. I confirmed
this by running kubectl get pvc and see a data-note-0 volume and data-note-1.

## 6. Delete note-0

```bash
kubectl delete pod note-0
kubectl get pods -l app=note -w
```

Wait until `note-0` is `Running` again, then press Ctrl-C.

The pod that is shutting down can show `Error` for a moment. That is the old container exiting. The replacement is the `note-0`.

```bash
kubectl exec note-0 -- cat /data/note.txt
kubectl exec note-1 -- cat /data/note.txt
kubectl get pvc
```

**Q5.** After `note-0` was deleted and came back, what stayed the same? How do these volumes differ from the Deployment?

When we cat either note the same data is there. When I did kubectl get pvc, the same volumes are there and they have the same
age which shows that they were never deleted, only the node was deleted.
In just a deployment, something could happen where when you start the node back up that the volume has changed as the volumes
in a deployment aren't stateful.

## Cleanup

```bash
kubectl delete -f stateful.yml
kubectl delete pvc data-note-0 data-note-1
```

(Note: Deleting the StatefulSet alone does not delete its volumes)
