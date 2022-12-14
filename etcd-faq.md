# ETCD FAQ

**IMPORTANT** Following recent student comments about the exam backup/restore questions,  this FAQ is currently under review. Once we ascertain what is actually required,  we will update this page, and the labs accordingly. 

`etcd` is a distributed key-value store, kind of similar to a NoSQL database. It is the database backend chosen by the Kubernetes project for the storage of cluster state. It is a separate open source project that is not maintained by the Kubernetes developers, but no doubt they have some input on its development.

* [What is ETCDCTL_API=3 all about?](#what-is-etcdctlapi3-all-about)
* [When do I use --endpoints?](#when-do-i-use---endpoints)
* [How do I make a backup?](#how-do-i-make-a-backup)
* [How do I restore a backup?](#how-do-i-restore-a-backup)
    * [kubeadm clusters with etcd running as a pod](#kubeadm-clusters-with-etcd-running-as-a-pod)
    * [Manually installed clusters](#manually-installed-clusters)
    * [Clusters with external etcd](#clusters-with-external-etcd)

**NOTE**: In the exam, you are advised to skip this question and come back to it at the end, as it is probably the hardest one and you don't want to waste time if there are easier questions further on!!

**NOTE ALSO**: On newer installations, the `etcdctl` command may not be present. If that's the case then the newer `etcdutl` should be. It has the same arguments.

## What is ETCDCTL_API=3 all about?

`etcdctl` supports two versions of the server's API. When making server calls, it defaults to version 2 of the API, and in version 2 some operations are either undefined, or have different arguments. Setting this environment variable tells `etcdctl` to use the V3 API, which is required for the snapshot functionality.

You may set the environment variable with each call...

```bash
ETCDCTL_API=3 etcdctl snapshot save ...
ETCDCTL_API=3 etcdctl snapshot restore ...
```

...or for the entire terminal session...

```bash
export ETCDCTL_API=3
etcdctl snapshot save ...
etcdctl snapshot restore ...
```

## When do I use --endpoints?

The `--endpoints` argument on `etcdctl` is used to tell it where to find the `etcd` server. If you are running the command on the same host where `etcd` service is running *and* there is only one instance of `etcd`, then you do not need to provide this argument, as it has a default value of `https://127.0.0.1:2379`.<br>This is the case in most labs, as your prompt will nearly always be on the controlplane node.

If you run `etcdctl` from a different workstation from where `etcd` is running, as may be the case in a production environment, *or* you need to use a non-standard port, then you need to provide this argument to say where the `etcd` server is.

* Same host, different port: `--endpoints https://127.0.0.1:port`
* Remote host: `--endpoints https://host-ip:port`

## How do I make a backup?

In order to take a backup, there are several required arguments, as we need to authenticate with the server to pull any data from it. These are

* `--cacert` - Path the the `etcd` CA certificate
* `--cert` - Path to the `etcd` server certificate
* `--key` - Path to the `etcd` server private key
* `--endpoints` - Only required in certain circumstances. See [this FAQ question](#when-do-i-use---endpoints).

On `kubeadm` clusters such as the labs, killer.sh or the exam, these files should be found in `/etc/kubernetes/pki/etcd` on the control plane node, which is where you should run `etcdctl` unless instructed otherwise. A question may also tell you which certs to use, in which case adjust the following accordingly.

Sample backup command

```bash
ETCDCTL_API=3 etcdctl snapshot save \
    --cacert /etc/kubernetes/pki/etcd/ca.crt \
    --cert /etc/kubernetes/pki/etcd/server.crt \
    --key /etc/kubernetes/pki/etcd/server.key \
    /opt/snapshot-pre-boot.db
```

## How do I restore a backup?

Normally you will restore this to another directory, and then point the `etcd` service at the new location. For restores, the certificate and endpoints arguments are not required, as all we are doing is creating files in directories and not talking to the `etcd` API, so the only argument required is `--data-dir` to tell `etcdctl` where to put the restored files.

What needs to be done following the restore to get `etcd` to use it depends on how the cluster is deployed.

Sample restore command

```bash
ETCDCTL_API=3 etcdctl snapshot restore \
    --data-dir /var/lib/etcd-from-backup \
    /opt/snapshot-pre-boot.db
```

### kubeadm clusters with etcd running as a pod

This is known as "stacked etcd"

It is a change to a single line in the manifest.

Determine that the cluster is configured this way by executing the following and determining that `etcd` is in the results

```
kubectl get pods -n kube-system
```

Now do the following on the control node.

1. "Stop" the kubernetes services by moving their manifests
```bash
cd /etc/kubernetes/manifests
mv kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml ..
```
2. Edit the manifest file for `etcd` on the controlplane node. This is found in <br>`/etc/kubernetes/manifests/etcd.yaml`.
1. Scroll down to the `volumes` section and find the volume that describes the data directory (see below).
1. Edit the `hostPath.path` of the volume with name `etcd-data` from `/var/lib/etcd` to `/var/lib/etcd-from-backup` (or whichever directory you used for the restore command). Note that you *do not* change the `--data-dir` command line argument to `etcd` in the container's command specification.
1. Restart the other services by moving their manifests back (this assumes you are still cd'ed to `/etc/kubernetes/manifests`)
```
mv ../*.yaml .
```
6. Wait about a minute for everything to reload.

This is the section in `etcd.yaml` that you need to find...

```yaml
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd    # <- change this
      type: DirectoryOrCreate
    name: etcd-data
```

Why does this work? You need to remember how mounting volumes in containers works.

* `volumes.hostPath.path` which you edited above specifies the directory on the node (host) where the data is stored. This is the actual location of where the backup was restored to.
* `containers.volumeMounts.mountPath` specifies the directory *inside* the container where the host data is mounted. We haven't changed that. From the `etcd` container's point of view, the directory is still `/var/lib/etcd`.

And note...

```yaml
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://10.40.10.9:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd                  # <- DO NOT CHANGE THIS!
    - --initial-advertise-peer-urls=https://10.40.10.9:2380
    - --initial-cluster=controlplane=https://10.40.10.9:2380

```

### Manually installed clusters

For clusters where `etcd` is deployed as an operating system service (like in [Kubernetes the hard way](https://github.com/mmumshad/kubernetes-the-hard-way)), you have to edit the command lime parameters to `etcd` by editing its service unit file. This is normally found in<br>`/etc/systemd/system/etcd.service`

1. Stop control plane services (`kube-apiserver`, `kube-controller-manager`, `kube-scheduler`) using `systemctl`
1. Edit the service unit file
1. Change the `--data-dir` argument to point to the new directory (e.g `/var/lib/etcd-from-backup`)
1. Note when editing in the service unit file whether a specific user account is being used to run `etcd`. If it is, you need to set the owner of the restored directory structure to this user with `chown -R`
1. Restart the services

```bash
sudo systemctl daemon-reload
sudo systemctl restart etcd kube-apiserver kube-controller-manager kube-scheduler
```

### Clusters with external etcd

Some people have suggested that this is how the exam is set up, as in you have to do it all from the exam node and *not* the control node for the target cluster. It requires a bit more work!

On the node where you are instructed to do the task, first determine how many `etcd` processes there are

```
sudo ps -x | grep bin/etcd
```

If there's more than one, you need to identify the correct one! In the output of the above command, you should be able to see the `--listen-client-urls` argument. Note down the port numbers from this URL for each. In the exam, use the `mousepad` application to take notes.

Go onto the control node for the target cluster and run

* If it is a kubeadm cluster (api server is running as a pod)

    ```
    grep etcd-servers /etc/kubernetes/manifests/kube-apiserver.yaml
    ```

* If it is not kubeadm, and api server is an operating system service

  ```
  sudo ps -x | grep apiserver
  ```

  Find `--etcd-servers` in the argument list and note the port number. You need to match that with one of the port numbers you've noted down. That will get you the correct `etcd` process.

Log out of the control node and return to the node running `etcd`.

Next, find the unit file for the correct `etcd` service. The following will give you the file *names*.

```
systemctl list-unit-files | grep etcd
```

> Output

```bash
etcd-1.service                               enabled         enabled
etcd-2.service                               enabled         enabled
```

To locate these files, run the following on each filename returned by the command above

```
sytemctl cat XXXX.service
```

...where `XXXX.service` is e.g. `etcd-1.service`. This will show the file content with a comment above it which is the location of the unit file.

> Output

```bash
# /etc/systemd/system/etcd-1.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
--- truncated---
  --listen-client-urls https://192.168.56.11:2379
```

Find the correct one by examining each identified unit file and choose the one that has the matching port number for the `--listen-client-urls` argument. You will need to edit this later.

Now do the backup and specify `--endpoint CLIENT_URL` where `CLIENT_URL` is the URL from the `--listen-client-urls` in the identified unit file.

Next do the restore using whatever backup file you are instructed to use.

Finally edit the identified unit file

1. Change the `--data-dir` argument to point to the new directory (e.g `/var/lib/etcd-from-backup`)
1. Note when editing in the service unit file whether a specific user account is being used to run `etcd`. If it is, you need to set the owner of the restored directory structure to this user with `chown -R`
1. Restart the service

```
sudo systemctl daemon-reload
sudo systemctl restart XXXX.service
```

where `XXXX.service` is the filename (not path) of the service unit file you have edited.

Please also try the following lab for practice of external etcd

> https://kodekloud.com/topic/practice-test-backup-and-restore-methods-2-2/
