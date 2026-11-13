# Task 2: FastAPI + MySQL with ConfigMap, Secret, PV/PVC and Ingress

For this task I deployed a small guestbook app on the same 3-node minikube cluster I used in Task 1. The app is FastAPI and it saves messages in MySQL. MySQL keeps its data on a PV/PVC so nothing is lost if the Pod dies. The app settings live in a ConfigMap, the MySQL passwords live in a Secret, and an Ingress lets me reach the app from the browser.

```
Browser -> Ingress Controller -> Ingress (webapp-ingress) -> Service webapp-api (80 -> 8000) -> 2 app Pods
                                                                                                   |
                                                                              Service mysql (3306) -> 1 MySQL Pod -> PVC -> PV (hostPath on minikube-m02)
```

The diagram from the task is here: [../task-session-2/images/architecture.png](../task-session-2/images/architecture.png)

## What I created, in simple words

- **Namespace `webapp-stack`**: one place for everything in this task, so it is easy to find and delete later.
- **Secret `mysql-secret`**: the MySQL passwords. Kubernetes stores them base64 encoded.
- **ConfigMap `webapp-config`**: normal settings like the page title and the MySQL address.
- **PV `mysql-pv`**: the actual storage, a folder on one node. A PV has no namespace.
- **PVC `mysql-pvc`**: the request for that storage. The MySQL Pod uses the PVC and the PVC is tied to the PV.
- **Deployment + Service `mysql`**: 1 MySQL Pod and a fixed address for it (`mysql:3306`).
- **Deployment + Service `webapp-api`**: 2 app Pods and one address that load-balances between them.
- **Ingress `webapp-ingress`**: sends traffic from outside to the `webapp-api` Service.

The files in this folder: `namespace.yaml`, `secret.yaml`, `configmap.yaml`, `pv-pvc.yaml`, `mysql.yaml`, `webapp.yaml`, `ingress.yaml`, and the `Screenshot/` folder with the numbered pictures used below.

---

## Step 1: Build the images and load them into minikube

The images `webapp-api:task2` and `webapp-mysql:task2` are not on Docker Hub, so I build them from the folders I was given (`webapp/` and `mysql/`). minikube has its own container runtime inside the nodes and can't see my local Docker images, so after building I have to load them into minikube.

```bash
cd task-session-2
minikube start
docker build -t webapp-api:task2 ./webapp
docker build -t webapp-mysql:task2 ./mysql
minikube image load webapp-api:task2
minikube image load webapp-mysql:task2
minikube image ls | grep task2
```

`minikube start` just starts the cluster if it was stopped. `minikube image ls` at the end is my check that both images are inside minikube. In the Deployments I set `imagePullPolicy: IfNotPresent`, otherwise Kubernetes would try to pull the images from the internet and fail.

![Images loaded](Screenshot/01-images-loaded.png)

## Step 2: Check the node names

The PV is a folder on one specific node, so I need the exact node name first.

```bash
kubectl get nodes -L kubernetes.io/hostname
```

The `-L` part adds a column with the hostname label, which is the label I use in the PV. I'm using `minikube-m02`.

![Node names](Screenshot/02-nodes-hostname.png)

## Step 3: Apply the YAML files

Order matters here. Namespace first, then the Secret, ConfigMap and PV/PVC, and the Deployments last, because the Pods read the Secret and the ConfigMap when they start.

```bash
kubectl apply -f namespace.yaml
kubectl apply -f secret.yaml
kubectl apply -f configmap.yaml
kubectl apply -f pv-pvc.yaml
kubectl apply -f mysql.yaml
kubectl apply -f webapp.yaml
kubectl apply -f ingress.yaml
```

There is one error in the screenshot. I typed `configmap.yaml\` with a backslash at the end, so kubectl couldn't find the file. I fixed the typo and ran it again.

![Apply the manifests](Screenshot/03-apply-manifests.png)

### What is in each file

**secret.yaml**: three keys, `root-password`, `app-user` and `app-password`. I used `stringData` so I can type the values as plain text and Kubernetes does the encoding.

**configmap.yaml**: five keys, `APP_TITLE`, `APP_ENV`, `MYSQL_HOST`, `MYSQL_PORT` and `MYSQL_DATABASE`. `MYSQL_HOST` is `mysql`, the name of the MySQL Service. Inside a namespace a Service name works like a DNS name, so the app finds MySQL without needing an IP.

`MYSQL_PORT` has to stay in the ConfigMap. Kubernetes automatically creates env variables for every Service, and for a Service called `mysql` it creates `MYSQL_PORT=tcp://10.x.x.x:3306`, which the app can't use. The value from the ConfigMap replaces it.

**pv-pvc.yaml**:
- The PV is 2Gi and uses a `hostPath` folder, `/data/webapp-mysql`.
- Both the PV and the PVC have `storageClassName: manual`. A PVC only binds to a PV with the same class. Without it minikube would use its default class `standard` and create a different volume.
- `persistentVolumeReclaimPolicy: Retain` keeps the data even if the PVC is deleted.
- `nodeAffinity` pins the PV to `minikube-m02`. A `hostPath` folder only exists on one node. Without the pin, MySQL could restart on another node, find an empty folder and look like it lost everything.
- The PVC points at the PV with `volumeName: mysql-pv`.

**mysql.yaml**:
- 1 replica with `strategy: Recreate`, so the old Pod is stopped before the new one starts. Two MySQL servers must never share the same data folder.
- Passwords come from the Secret (`secretKeyRef`) and the database name comes from the ConfigMap (`configMapKeyRef`).
- The PVC is mounted at `/var/lib/mysql`, where MySQL keeps its data.
- Readiness probe runs `mysqladmin ping`. Liveness probe waits 60 seconds first, because the first start takes a while.
- The Service is a ClusterIP on port 3306.

**webapp.yaml**:
- 2 replicas. The whole ConfigMap goes in with `envFrom`, and `MYSQL_USER` and `MYSQL_PASSWORD` come from the Secret.
- Readiness and liveness probes call `GET /health` on port 8000.
- The Service maps port 80 to the container port 8000.

**ingress.yaml**: `ingressClassName: nginx`, path `/` with `pathType: Prefix`, pointing at the `webapp-api` Service on port 80.

## Step 4: Install the Ingress Controller

An Ingress is just a set of rules. Something has to actually follow them, and that is the Ingress Controller, a Pod running nginx. I used the minikube addon.

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

The controller Pod is `Running` and the `nginx` IngressClass exists. The two `Completed` Pods are one-time jobs that set up certificates, so it is normal that they are not running.

![Ingress controller](Screenshot/04-ingress-controller.png)

## Step 5: Check that everything is correct

### 5.1 Secret and ConfigMap

```bash
kubectl get secret mysql-secret -n webapp-stack
kubectl get configmap webapp-config -n webapp-stack
```

The Secret has 3 keys and the ConfigMap has 5.

![Secret and ConfigMap](Screenshot/05-secret-configmap.png)

### 5.2 PV and PVC

```bash
kubectl get pv mysql-pv
kubectl get pvc mysql-pvc -n webapp-stack
```

Both are `Bound` and 2Gi. The PV shows `webapp-stack/mysql-pvc` as its claim, so they found each other.

![PV and PVC](Screenshot/06-pv-pvc.png)

### 5.3 Deployments, Pods and Services

```bash
kubectl get deploy -n webapp-stack
kubectl get pods -n webapp-stack -o wide
kubectl get svc,endpoints -n webapp-stack
```

`mysql` is 1/1 and `webapp-api` is 2/2. The MySQL Pod is running on `minikube-m02`, which is the node with the data folder, so the `nodeAffinity` is doing its job. The endpoints show 1 IP for `mysql` and 2 IPs for `webapp-api`, so both Services found their Pods.

![Deployments and Pods](Screenshot/07-deployments.png)

### 5.4 Environment variables inside the app

```bash
kubectl exec -n webapp-stack deploy/webapp-api -- env | grep -E "APP_|MYSQL_"
```

`kubectl exec` runs a command inside a Pod. I can see the ConfigMap values (`APP_TITLE`, `MYSQL_HOST=mysql`) and the Secret values (`MYSQL_USER`, `MYSQL_PASSWORD`). `MYSQL_PORT` is `3306`, not the `tcp://...` one, so the ConfigMap won as expected.

![App environment](Screenshot/08-app-env.png)

### 5.5 Default data in MySQL

```bash
kubectl exec -n webapp-stack deploy/mysql -- sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT user,host FROM mysql.user; SELECT * FROM webapp.messages"'
```

The `webapp` user exists and the `messages` table has the 2 default messages. The init script of the MySQL image creates them on the first start.

![Default data](Screenshot/09-default-data.png)

### 5.6 Ingress

```bash
kubectl get ingress webapp-ingress -n webapp-stack
```

The class is `nginx` and the ADDRESS is filled in, so the controller picked it up.

![Ingress](Screenshot/10-ingress.png)

## Step 6: Open the app

My minikube runs with the Docker driver on Windows, so I can't just open the minikube IP in the browser. I used `port-forward` to the Ingress Controller and left it running in a second terminal.

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Then I opened <http://localhost:8080>. The page shows the title from the ConfigMap, the environment `dev`, which app Pod answered and which MySQL Pod it talked to. I posted a few messages in the guestbook and they were saved. If I refresh, the app Pod name sometimes changes, that is the Service balancing between the 2 Pods.

![App in the browser](Screenshot/11-app-in-browser.png)

## Step 7: Test the API with curl

```bash
curl http://localhost:8080/api/info
curl -X POST http://localhost:8080/api/messages -H "Content-Type: application/json" -d '{"author":"student","content":"hello"}'
curl http://localhost:8080/api/messages
```

`/api/info` shows `"version": "task2"`, the title, `"environment": "dev"` and the MySQL Pod name, which means the app really reached the database.

![api/info](Screenshot/12-api-info.png)

The POST gives back `id` 3 (1 and 2 are the default messages) and the MySQL Pod that wrote it. The GET returns my message plus the 2 defaults.

![Write and read messages](Screenshot/13-api-messages.png)

## Step 8: Does the data survive a restart?

This is the whole point of the PV. I deleted the MySQL Pod, the Deployment made a new one, and the new Pod mounted the same PVC.

```bash
kubectl delete pod -n webapp-stack -l app=mysql
kubectl get pods -n webapp-stack -l app=mysql -o wide
curl http://localhost:8080/api/messages
```

The Pod has a new name, it is on `minikube-m02` again, and my message with `id` 3 is still there.

![Data survives a restart](Screenshot/14-persistence.png)

---

## The problem I ran into: `Access denied for user 'webapp'`

### What I saw

Everything was deployed and the Pods looked fine, but the page showed an error where the MySQL server name should be, and the guestbook said "Database not ready yet":

```
error: (1045, "Access denied for user 'webapp'@'10.244.151.10' (using password: YES)")
```

![Access denied in the browser](Screenshot/15-issue-access-denied.png)

### My first guess (wrong, or at least not enough)

The MySQL image only creates the database and the `webapp` user on the very first start, when the data folder is empty. If the folder already has data, it skips that. So I guessed the folder had old data from an earlier run with other passwords. I checked the folder on the node and it was full of MySQL files:

```bash
minikube ssh -n minikube-m02 -- "sudo ls -la /data/webapp-mysql"
```

![Old data in the folder](Screenshot/16-issue-old-data-folder.png)

So I deleted the Deployment, deleted the folder and applied again. Same error.

```bash
kubectl delete -f mysql.yaml
minikube ssh -n minikube-m02 -- sudo rm -rf /data/webapp-mysql
kubectl apply -f mysql.yaml
```

![First wipe attempt](Screenshot/17-issue-first-wipe-attempt.png)

Looking back, I made two mistakes there. I deleted the folder while the old Pod might still have been shutting down, so I should have waited until `kubectl get pods` showed no MySQL Pod. And the new Pod showed `RESTARTS 1` only a few seconds after starting. A healthy Pod should not restart, so something was crashing and I didn't look into it yet.

### Finding the real cause

`kubectl logs` shows the current container, and by then that was already the second run. The crash happened in the previous container, so I used `--previous`:

```bash
kubectl logs deploy/mysql -n webapp-stack --previous
kubectl describe pod -n webapp-stack -l app=mysql | grep -A8 "Last State"
```

The important lines:

```
running /docker-entrypoint-initdb.d/01-setup.sh
/docker-entrypoint-initdb.d/01-setup.sh: /bin/bash^M: bad interpreter: No such file or directory
...
Last State: Terminated   Reason: Error   Exit Code: 126
```

The `^M` is a hidden `\r` character, which means Windows line endings (CRLF). Git on my machine has `core.autocrlf=true`, so it changed the init script from LF to CRLF when I checked it out. Linux then looked for a program literally called `/bin/bash\r`, which doesn't exist, so the script crashed with exit code 126. Because of that the `webapp` database and user were never created. MySQL restarted, kept the half-finished data, and that is why the app got `Access denied`. My YAML was fine, the problem was inside the image.

### Fix, part 1: line endings

I converted the script back to LF:

```bash
sed -i 's/\r$//' mysql/initdb/01-setup.sh
```

After rebuilding, the previous container's log showed a second, different error:

```
01-setup.sh: line 9: mysql_note: command not found
01-setup.sh: line 10: docker_process_sql: command not found
```

### Fix, part 2: file permissions

The official MySQL entrypoint handles `.sh` files in two ways. If the file is not executable, it sources it, meaning it loads the file into its own shell, and the helper functions `mysql_note` and `docker_process_sql` are available. If the file is executable, it runs it as a separate process, and those functions don't exist there, so it fails.

Docker on Windows copies files into the image as executable (755). On Linux the same file would normally be 644, which is why this only breaks on Windows. I fixed it in `task-session-2/mysql/Dockerfile` by setting the permission on the `COPY` line:

```dockerfile
COPY --chmod=644 initdb/01-setup.sh /docker-entrypoint-initdb.d/01-setup.sh
```

> **Side note: the course repo already had a fix for this.**
> When I finished, I went and checked the latest commits on the original course repo to see if anything changed in Task 2. It only edits `task-session-2/README.md` and adds one line to Step 1, before the `docker build` commands:
>
> ```bash
> chmod -x mysql/initdb/01-setup.sh
> ```
>
> So the course authors hit the same executable-bit problem I did, and their answer is to remove the executable bit before building. I wanted to know if that would have saved me, so I tried it on my own machine. I copied the `mysql` folder, ran `chmod -x` on the script in Git Bash, and checked it:
>
> ```
> before chmod -x:  -rwxr-xr-x 1 Sallam 197121 703 Nov 13 02:41 01-setup.sh
> after  chmod -x:  -rwxr-xr-x 1 Sallam 197121 703 Nov 13 02:41 01-setup.sh
> ```
>
> Nothing changed. My project is on an NTFS drive, which doesn't really have an executable bit, so Git Bash can't remove it. I built the image anyway and looked inside it, and the script was still `-rwxr-xr-x`. That means on my Windows setup the course fix does nothing, and I would have got the same `command not found` error.
>
> That is why I put `--chmod=644` in the Dockerfile instead. Docker sets the permission itself while building, so it doesn't matter what the file system on my laptop says. I also still needed the `sed` command for the CRLF problem, which the course fix doesn't touch at all. So on my machine I needed both fixes.

### Rebuild and redeploy

```bash
docker build -t webapp-mysql:task2 ./mysql
minikube image load webapp-mysql:task2 --overwrite=true

kubectl delete -f mysql.yaml
kubectl get pods -n webapp-stack -l app=mysql          # wait until: No resources found
minikube ssh -n minikube-m02 -- "sudo rm -rf /data/webapp-mysql/*"
minikube ssh -n minikube-m02 -- "sudo ls -la /data/webapp-mysql"   # should be empty
kubectl apply -f mysql.yaml

kubectl rollout restart deploy/webapp-api -n webapp-stack
```

I rebuilt and reloaded the image because the cluster still had the old one with the broken script. This time I waited for the old Pod to disappear before emptying the folder, so MySQL saw an empty folder and ran the init script. At the end I restarted the app Pods so they reconnect.

![Rebuild the image](Screenshot/18-fix-rebuild-image.png)

![Wipe the data and redeploy MySQL](Screenshot/19-fix-wipe-and-redeploy.png)

Now the log shows the init script being sourced and the database and user being created, and the Pod has `RESTARTS 0`:

```bash
kubectl logs deploy/mysql -n webapp-stack | grep -E "Initializing|initdb|Creating database|ready for connections"
```

![MySQL initialized correctly](Screenshot/20-fix-mysql-logs.png)

After that, all the checks from Steps 5 to 8 passed.

### What I take from this

- `Access denied` was only the symptom. The real cause was in the MySQL logs, so next time I'll check `kubectl logs` and `kubectl logs --previous` much earlier.
- If a new Pod already has `RESTARTS` above 0, I should stop and read `--previous` and `kubectl describe pod` before guessing.
- Files from Windows can break Linux containers, through CRLF line endings or the executable bit. A `.gitattributes` file with `*.sh text eol=lf` would stop the CRLF part.
- MySQL creates its database and user only on the first start. If the data folder already has data, changing the Secret does nothing. To start clean: delete the Deployment, wait until the Pod is gone, then empty the folder.
- When kubectl says `connection refused` on `127.0.0.1`, the cluster is probably just stopped, so `minikube start` fixes it.

## Commands I kept using

```bash
kubectl get all -n webapp-stack                          # everything in the namespace
kubectl describe pod <pod> -n webapp-stack               # events and details of a Pod
kubectl logs deploy/mysql -n webapp-stack                # logs of the current container
kubectl logs deploy/mysql -n webapp-stack --previous     # logs of the container before the restart
kubectl rollout restart deploy/webapp-api -n webapp-stack
```

## Clean up

I didn't delete anything, because Task 3 starts from this stack and these YAML files.
