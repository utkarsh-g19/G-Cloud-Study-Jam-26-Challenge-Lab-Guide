# GSP319: Build a Website on Google Cloud (Challenge Lab)

A complete walkthrough with the mistakes people usually make and how to fix them.

| | |
|---|---|
| **Lab ID** | GSP319 |
| **Official title** | Build a Website on Google Cloud: Challenge Lab |
| **Scoring** | 100 points across 6 checkpoints |
| **Time** | 90 minutes of access, about 25 minutes of actual work (a cluster problem can cost extra time) |
| **Official page version used** | Manual and tests last updated August 4, 2026 |
| **Where you work** | Cloud Shell and a browser tab (no notebook this time) |

| Checkpoint | Points |
|---|---|
| Task 1: Download the monolith code and build your container | 10 |
| Task 2: Create a GKE cluster and deploy the monolith | 20 |
| Task 3: Build Orders and Products container images | 20 |
| Task 4: Deploy the Orders and Products containers | 20 |
| Task 5: Configure the frontend (no checkpoint, but Tasks 6 and 7 depend on it) | none |
| Task 6: Build the frontend container | 10 |
| Task 7: Deploy the frontend container | 20 |

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Mistake to avoid** show a real error people usually run into, and the fix. Blocks marked **Note** are things the official page doesn't spell out.
>
> **Values written in `UPPERCASE` are placeholders.** Your lab assigns its own image names, container names, cluster name and region. The public version of the lab page leaves these blank, and they only appear inside your running lab. Always copy them from your own lab page. In the example run they were `fancy-monolith-351`, `fancy-orders-964`, `fancy-products-304`, `fancy-frontend-538`, cluster `fancy-cluster-159`, and region `us-central1`. Yours will differ.

---

# Part 1: Conceptual

## What this lab is about

A company runs an online shop called **Fancy Store** as one large program, called a **monolith**. In this lab you break it into smaller independent services (**microservices**), package each into a container, run them on **Google Kubernetes Engine (GKE)**, and then point a new frontend at them. In order, you:

1. Build the old monolith into a container image and push it to Google's registry.
2. Create a Kubernetes cluster and run the monolith on it.
3. Build separate images for the **Orders** and **Products** services.
4. Run those two services in the cluster.
5. Rewire the website's frontend to talk to the new services.
6. Build and deploy the new frontend.

Challenge labs give you the goal but not the exact commands, so you apply what the course taught.

## Important concepts, explained simply

**Monolith vs microservices.** A monolith is one program doing everything (frontend, orders, products). Microservices split each job into its own small program. They can be updated, scaled and fixed separately.

**Container and image.** A *container* is a sealed box holding a program and everything it needs to run. An *image* is the template for making that box. You build an image once and can run it anywhere.

**Dockerfile.** The recipe that tells the build how to make an image. Each service folder contains one. The build command looks for it in the **current directory**, so where you stand matters.

**Cloud Build and `gcloud builds submit`.** A Google service that builds images for you. `gcloud builds submit --tag ... .` uploads the current folder, builds the image from its Dockerfile, and stores it. The `.` at the end means "this folder".

**Container registry (`gcr.io/...`).** The warehouse where built images are stored. The path looks like `gcr.io/PROJECT_ID/IMAGE_NAME:VERSION`.

**Tag/version (`:1.0.0`).** A label for one version of an image. The lab asks for exactly `1.0.0`, so use it everywhere.

**GKE cluster and nodes.** A *cluster* is a group of machines (*nodes*) that Kubernetes manages for you. You tell Kubernetes what to run and it decides where.

**Region vs zone.** A *region* (like `us-central1`) is a geographic area made of several *zones* (like `us-central1-a`). This matters for node counts (see Task 2).

**Deployment and pod.** A *deployment* says "run this image and keep it running." A *pod* is a running copy of your container.

**Service of type LoadBalancer.** A pod's address isn't reachable from the internet. A *service* gives it a stable front door. `LoadBalancer` asks Google to create a public **external IP**.

**Port and target-port.** `--port 80` is the port the outside world uses. `--target-port 8080` is the port the app inside the container listens on. The service connects the two.

**`kubectl`.** The command-line tool for talking to Kubernetes.

**Quotas.** Each project has limits (number of IP addresses, disk space and so on). A request that would go over a limit fails with a "quota exceeded" error.

**`.env` file and React build.** The React frontend reads service addresses from a `.env` file at build time. Changing the file does nothing until you rebuild with `npm run build`, then rebuild the container image.

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**.
2. Click **Open Google Cloud console**. Sign in with the **Username and Password from the lab panel**, accept the terms and skip the recovery page.
3. Check that the project selector shows a `qwiklabs-gcp-...` project.
4. **Write down from your lab page:** the region (or zone), the cluster name, and the image and container names for the monolith, orders, products and frontend.
5. Don't click **End Lab** until all six checkpoints are green.

> **Note: naming and resources.** The official page asks you to use `e2-medium` and not to oversize anything, and warns that excessive resource use can get the project terminated. Don't create extra clusters or machines.

> **Mistake to avoid: mixing student accounts.** During this lab the browser can sign you out, and the account chooser may list several student accounts from different labs. Only one belongs to the lab that's running.
>
> **Fix.** Pick the account whose email matches the **Username in your current lab panel**, and use a fresh private window with only that account signed in. Once signed in, open Cloud Shell again and click **Authorize**. Then reconnect to your cluster (see Task 3, step 3.0).

---

## Task 1: Download the monolith code and build your container (10 points)

### 1.1 Open Cloud Shell

Click **Activate Cloud Shell** (`>_`), then **Continue** and **Authorize** if asked. Wait for the `$` prompt.

### 1.2 Clone the repository and run setup

Run these one at a time:

```bash
cd ~
git clone https://github.com/googlecodelabs/monolith-to-microservices.git
cd ~/monolith-to-microservices
./setup.sh
```

`setup.sh` takes a few minutes. It installs packages, then builds the React app.

> **Note: harmless noise.** The output contains `76 vulnerabilities`, `npm warn install-scripts`, a Node `DeprecationWarning` and `caniuse-lite is outdated`. They don't matter for the lab.
>
> **Don't** run `npm audit fix` or `npm audit fix --force`. They can change package versions and break the build. **Don't** press Ctrl+C. Wait until the `$` prompt returns. The last stage, "Building React app and placing into sub projects...", takes one to three minutes.

### 1.3 Update Node.js

The lab page has a code box for this step (Task 1, step 3). On the public page the command appears blank, so **copy it from your own lab page**, paste it into Cloud Shell and press Enter. Then check:

```bash
node -v
```

### 1.4 Build and push the monolith image

Move into the monolith folder, and run each command **separately**:

```bash
cd ~/monolith-to-microservices/monolith
```

```bash
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/MONOLITH_IMAGE_NAME:1.0.0 .
```

Replace `MONOLITH_IMAGE_NAME` with the name from your lab page (the example used `fancy-monolith-351`). Keep the final dot. If it asks to enable Cloud Build or Artifact Registry APIs, answer **y**. Wait for `STATUS: SUCCESS`.

> **Mistake to avoid: two problems at once.**
>
> 1. People often paste the `cd ...` and `gcloud builds submit ...` lines together as one pasted line, and Cloud Shell prints `-bash: cd: too many arguments`. Pressing Ctrl+C mid-build also causes problems (`Command killed by keyboard interrupt`).
> 2. The next attempt failed with `ERROR: (gcloud.builds.submit) You do not currently have an active account selected.` Cloud Shell had lost its login, usually because the **Authorize** pop-up was dismissed or hidden.
>
> **Fix, in order:**
> 1. Run `gcloud auth list`. If it shows "No credentialed accounts", the login is gone.
> 2. Close the Cloud Shell terminal (the **X** by the project name), click **Activate Cloud Shell** (`>_`) again, and click **Authorize** when the pop-up appears. (Allow pop-ups for the console if it's blocked.)
> 3. If that doesn't work, run `gcloud auth login --no-launch-browser`, open the link it prints, sign in with the **current lab's** student account, and paste the verification code back.
> 4. Confirm with `gcloud auth list` and `echo $GOOGLE_CLOUD_PROJECT`.
> 5. Paste the `cd` and the `gcloud builds submit` commands **one at a time** and **never press Ctrl+C** during a build.

Verify the image exists:

```bash
gcloud container images list
```

### 1.5 Check progress

Click **Check my progress** under Task 1 (10 points).

---

## Task 2: Create a GKE cluster and deploy the monolith (20 points)

### 2.1 Create the cluster

The lab gives you a cluster name and a location, and asks for 3 nodes of `e2-medium`.

First, see whether your location is a **zone** (ends in a letter, like `us-central1-a`) or a **region** (like `us-central1`). In the example run it was a **region**.

If it's a zone:

```bash
gcloud container clusters create CLUSTER_NAME --zone ZONE --num-nodes 3 --machine-type e2-medium
```

If it's a **region**:

```bash
gcloud container clusters create CLUSTER_NAME --region REGION --num-nodes 1 --machine-type e2-medium
```

> **Mistake to avoid: a quota error from the node count.** People often create a regional cluster with `--num-nodes 3`. For a regional cluster, `--num-nodes` means *nodes per zone*, and a region has 3 zones, so that asks for **9 nodes**. The command fails with:
>
> ```
> Quota 'IN_USE_ADDRESSES' exceeded.  Limit: 8.0 in region us-central1
> Quota 'SSD_TOTAL_GB' exceeded.  Limit: 500.0 in region us-central1
> ```
>
> **Fix, in order:**
> 1. List the failed cluster: `gcloud container clusters list --region us-central1`.
> 2. Delete it: `gcloud container clusters delete CLUSTER_NAME --region us-central1 --quiet`. Wait for it to finish.
> 3. Confirm everything is gone, because leftover instances still hold quota: `gcloud container clusters list --region us-central1` and `gcloud compute instances list`. Wait and recheck if `gke-...` instances are still listed.
> 4. Recreate with `--num-nodes 1`. That gives 1 node in each of 3 zones, which is the 3 nodes the lab wants.
> 5. If `SSD_TOTAL_GB` comes back, add `--disk-size 50`.
>
> The final cluster should show `STATUS: RUNNING` and `NUM_NODES: 3`.

Cluster creation takes 5 to 10 minutes. Don't press Ctrl+C.

### 2.2 Connect and check

```bash
gcloud container clusters get-credentials CLUSTER_NAME --region REGION
kubectl get nodes
```

(Use `--zone ZONE` here if your location is a zone.) You should see 3 nodes with status `Ready`.

### 2.3 Deploy and expose the monolith

Run these one at a time:

```bash
kubectl create deployment MONOLITH_CONTAINER_NAME --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/MONOLITH_IMAGE_NAME:1.0.0
```

```bash
kubectl expose deployment MONOLITH_CONTAINER_NAME --type=LoadBalancer --port 80 --target-port 8080
```

In the example run the container name matched the image name. **Check** the Task 2 section of your lab page for the container name, because the checker compares it.

> **Mistake to avoid.** If you leave a placeholder such as `MONOLITH_IMAGE_NAME` inside a command, Kubernetes rejects it with a confusing error (see Task 7 for the real example).

### 2.4 Get the external IP and test

```bash
kubectl get service
```

`EXTERNAL-IP` shows `<pending>` at first. Rerun every 30 seconds until it shows an address. Then open `http://EXTERNAL-IP` in a browser. You should see the Fancy Store page. Use port 80, so don't add `:8080`.

Check the pod: `kubectl get pods` should show `Running`. If it shows `ImagePullBackOff`, the image name or tag is wrong. Compare with `gcloud container images list` and `kubectl describe pod POD_NAME`.

### 2.5 Check progress

Click **Check my progress** under Task 2 (20 points).

---

## Task 3: Build Orders and Products container images (20 points)

### 3.0 Reconnect after a Cloud Shell reset

If you reopen Cloud Shell (for example, after a sign-out), your session has lost its cluster connection and its folder. Run:

```bash
gcloud auth list
echo $GOOGLE_CLOUD_PROJECT
gcloud container clusters get-credentials CLUSTER_NAME --region REGION
kubectl get nodes
```

You should see the student account, your project ID, and 3 `Ready` nodes.

### 3.1 Build Orders

```bash
cd ~/monolith-to-microservices/microservices/src/orders
```

```bash
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/ORDERS_IMAGE_NAME:1.0.0 .
```

Wait for `STATUS: SUCCESS` (it took about 40 seconds in the example run). The example image was `fancy-orders-964`.

### 3.2 Build Products

```bash
cd ~/monolith-to-microservices/microservices/src/products
```

```bash
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/PRODUCTS_IMAGE_NAME:1.0.0 .
```

> **Mistake to avoid: two errors back to back on the Products build.**
>
> 1. `ERROR: (gcloud.builds.submit) HTTPError 401: Invalid Credentials. This command is authenticated as student-...@qwiklabs.net`. The login token had gone stale after the sign-out, even though it named the correct account.
> 2. After reopening Cloud Shell, the next attempt printed `Dockerfile required when specifying --tag`. A fresh Cloud Shell starts in your **home folder** (`~`), not in the folder you were in before, and the build looks for a Dockerfile in the current folder.
>
> **Fix, in order:**
> 1. Close and reopen Cloud Shell and click **Authorize** to refresh the login. (Or use `gcloud auth login --no-launch-browser`.)
> 2. Run `gcloud auth list` to confirm the account.
> 3. Run `cd ~/monolith-to-microservices/microservices/src/products` and check the prompt shows that path.
> 4. Run `ls` and confirm you can see `Dockerfile`.
> 5. Run the `gcloud builds submit ...` command again.
>
> Rule: **before every build, check the prompt shows the right folder.**

> **Note: image names differ by service.** The example names ended in different numbers (`fancy-orders-964` and `fancy-products-304`). That looks odd but was correct, because each name comes from the lab page. Don't "fix" one to match the other. Copy each one from the page.

### 3.3 Verify

```bash
gcloud container images list
```

You should see all three images (monolith, orders, products).

### 3.4 Check progress

Click **Check my progress** under Task 3 (20 points). This checkpoint is about the images being built. Deploying comes in Task 4.

---

## Task 4: Deploy the Orders and Products containers (20 points)

Orders listens on **8081** and Products on **8082**, both exposed externally on port 80. Run each command separately.

**Orders:**

```bash
kubectl create deployment ORDERS_CONTAINER_NAME --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/ORDERS_IMAGE_NAME:1.0.0
```

```bash
kubectl expose deployment ORDERS_CONTAINER_NAME --type=LoadBalancer --port 80 --target-port 8081
```

**Products:**

```bash
kubectl create deployment PRODUCTS_CONTAINER_NAME --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/PRODUCTS_IMAGE_NAME:1.0.0
```

```bash
kubectl expose deployment PRODUCTS_CONTAINER_NAME --type=LoadBalancer --port 80 --target-port 8082
```

Replace every `UPPERCASE` word with the real value before pressing Enter.

Then:

```bash
kubectl get service
```

Wait until both new services have an `EXTERNAL-IP` and **write both IPs down**. You need them in Task 5. They belong to the orders and products services, not the monolith.

Test them:

```bash
curl http://ORDERS_EXTERNAL_IP/api/orders
curl http://PRODUCTS_EXTERNAL_IP/api/products
```

Each should return JSON. Check pods with `kubectl get pods` (all `Running`).

If you made a typo, delete and redo it:

```bash
kubectl delete service NAME
kubectl delete deployment NAME
```

Click **Check my progress** under Task 4 (20 points).

---

## Task 5: Reconfigure the frontend (no checkpoint)

This task has no checkpoint, but skipping it or getting it wrong makes Tasks 6 and 7 fail quietly. The frontend gets its service addresses from a `.env` file, and the build bakes them in.

1. Get the two IPs again if needed:

```bash
kubectl get service
```

2. Open the file:

```bash
cd ~/monolith-to-microservices/react-app
nano .env
```

It starts with `localhost` addresses:

```
REACT_APP_ORDERS_URL=http://localhost:8081/api/orders
REACT_APP_PRODUCTS_URL=http://localhost:8082/api/products
```

3. Change it to read exactly:

```
REACT_APP_ORDERS_URL=http://ORDERS_IP_ADDRESS/api/orders
REACT_APP_PRODUCTS_URL=http://PRODUCTS_IP_ADDRESS/api/products
```

- Use the real IPs, with no `<` or `>`.
- **No port numbers.** Both services are on port 80.
- Don't swap the IPs. Orders IP on the orders line, Products IP on the products line.
- Keep `/api/orders` and `/api/products` at the end.

4. Save: **Ctrl+O**, **Enter**, **Ctrl+X**.

5. **Verify** (this step is worth the extra time):

```bash
cat .env
```

6. Rebuild:

```bash
npm run build
```

It takes one to three minutes. Wait for the prompt to return, and ignore the `caniuse-lite` and deprecation warnings. If you see `react-scripts: not found` or missing modules, run `npm install` first, then `npm run build` again.

---

## Task 6: Build the frontend container (10 points)

Go to the frontend service folder:

```bash
cd ~/monolith-to-microservices/microservices/src/frontend
ls
```

You should see `Dockerfile`. Then:

```bash
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/FRONTEND_IMAGE_NAME:1.0.0 .
```

The lab warns this may take a few minutes. Wait for `STATUS: SUCCESS` (in the example run it took 32 seconds and the image was `fancy-frontend-538`). Verify with `gcloud container images list`, which should show four images. Click **Check my progress** under Task 6 (10 points).

If you hit `Dockerfile required` or `401 Invalid Credentials`, use the fixes from Task 3.2.

---

## Task 7: Deploy the frontend container (20 points)

The frontend listens on **8080** and is exposed externally on port 80.

```bash
kubectl create deployment FRONTEND_CONTAINER_NAME --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/FRONTEND_IMAGE_NAME:1.0.0
```

> **Mistake to avoid: a placeholder left in the command.** People often type the real deployment name but leave `FRONTEND_IMAGE_NAME` in the image path. Kubernetes tried to turn that text into a container name and rejected it:
>
> ```
> Deployment.apps "fancy-frontend-538" is invalid: spec.template.spec.containers[0].name:
> Invalid value: "FRONTEND-IMAGE-NAME-c729z": a lowercase RFC 1123 label must consist of
> lower case alphanumeric characters or '-' ...
> ```
>
> The error looks like a naming problem, but the cause is the leftover placeholder (uppercase letters and underscores). Nothing was created.
>
> **Fix.** Put the **real image name** in the path and rerun:
>
> ```bash
> kubectl create deployment fancy-frontend-538 --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-frontend-538:1.0.0
> ```
>
> Use your own names. The deployment name must match the container name on your lab page, and the image path must match the image you built. Before pressing Enter, read the whole command and check **no uppercase placeholder is left**.

Then expose it:

```bash
kubectl expose deployment FRONTEND_CONTAINER_NAME --type=LoadBalancer --port 80 --target-port 8080
kubectl get service
```

Wait for the frontend's `EXTERNAL-IP`. Don't confuse it with the monolith, orders or products IPs. Check pods with `kubectl get pods`, then open `http://FRONTEND_EXTERNAL_IP` in your browser. You should see the Fancy Store homepage, with working Products and Orders pages.

> **If the Products or Orders pages are empty or erroring,** the `.env` file from Task 5 probably has a wrong IP. Check `cat ~/monolith-to-microservices/react-app/.env`. If an IP is wrong, fix it, run `npm run build` in `react-app`, rebuild the frontend image (Task 6), and redeploy.

Click **Check my progress** under Task 7 (20 points).

---

## Finish

1. Confirm all six checkpoints are green and the score reads **100**.
2. Only then click **End Lab** and confirm. Ending the lab wipes everything and you can't resume it.

If a checkpoint stays red, wait 20 seconds and click again, then check the exact names and the `1.0.0` tag against your lab page.

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order, commands and checkpoints. It adds the warnings and fixes below.

| Official lab | This guide |
|---|---|
| Task 1: clone repo, run `setup.sh`, update Node.js, build the monolith image with the tag from the page | Same steps. Adds the harmless-warnings note, the "copy the Node command from your page" reminder, and the lost-login and paste-one-at-a-time fixes. |
| Task 2: create the cluster (3 nodes, `e2-medium`), deploy the monolith, expose port 80 to 8080 | Same steps. Adds the region-vs-zone node count rule and the quota error fix. |
| Task 3: build Orders and Products images | Same steps. Adds the reconnect-after-reset step, the stale 401 login fix, and the "check your folder" rule that prevents the Dockerfile error. |
| Task 4: deploy Orders (8081) and Products (8082) | Same steps. Adds the reminder to write both IPs down and test with `curl`. |
| Task 5: edit `.env` with the two IPs and run `npm run build` | Same steps. Adds the exact format, the `cat .env` check and what to do if the build fails. |
| Task 6: build the frontend image | Same steps. Adds the folder check and the verification command. |
| Task 7: deploy the frontend (8080 to 80) | Same steps. Adds the leftover-placeholder error and the fix. |

> **What the official page doesn't show.** The public page leaves out your region or zone, the cluster name, and the image and container names, because your running lab fills them in. The values in this guide are the ones from the example run. Copy yours from your own lab page.
