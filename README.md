# Plakar Kubernetes Operator Demo

#### Here's what's covered in this demo and provided by this repo:

- Create a demo cluster
- Run a simple hello world app in the cluster
- Run WordPress with MariaDB in the cluster
- Use the [Plakar Kubernetes Operator](https://www.plakar.io/docs/control-plane/infrastructure-as-code/kubernetes-operator/) to back up and restore file uploads and a database
    - They will get backed up to an S3 bucket hosted on AWS

#### If you plan to follow along:

- Have a [Plakar Control Plane](https://www.plakar.io/docs/control-plane/intro/overview/) (PCP) running. It can be hosted anywhere
- Have an AWS account with administrator access.

## Getting Started

### Cloning this repo

```sh
git clone https://github.com/PlakarKorp/kubernetes-operator-demo-aws-eks.git
cd kubernetes-operator-demo-aws-eks
```

### Important files

In an attempt to keep this demo easy to understand we'll be using raw Kubernetes
config files instead of using Kustomize, Helm or other templating tools.

With that said, certain values within certain files will need to change
depending on where your cluster is running (kind, EKS, etc.) and for things
like defining your S3 bucket name and other details.

To help manage that, we're going to use `envsubst` which is a standard tool to
let you template any files with environment variables. You don't need to know a
lot about it, but if you're curious, here's a [blog post and video
tutorial](https://nickjanetakis.com/blog/using-envsubst-to-merge-environment-variables-into-config-files)
going over how `envsubst` works.

For us, all this means is instead of running `kubectl apply -f ...` or `kubectl
delete -f ...` commands we'll be running `./run kubectl:apply` and `./run
kubectl:delete` commands and those little shortcuts will handle wiring up the
`envsubst` command for us. Other `kubectl` commands can be run directly, or if
you prefer you can do `./run kubectl` as a convenience.

### Run script

This is a little shell script to help run long commands. You can run `./run` to
get a full list of commands. I'd encourage you to check it out in your code
editor of choice.

For example, here's how `kubectl:apply` and `kubectl:delete` are defined:

```sh
_kubectl:file() {
  local action="${1}"
  local config_path="${2}"
  shift 2

  envsubst <"${config_path}" | kubectl "${action}" "${@}" -f -
}

kubectl:apply() {
  _kubectl:file apply "${@}"
}

kubectl:delete() {
  _kubectl:file delete "${@}"
}
```

### Environment file

There's a `.env.example` file committed to this repo, **please run `cp -n
.env.example .env` to copy it and then modify the `.env` file at your
discretion**.

It stores a few environment variables that you can tweak when it comes to
customizing things like the cluster name, Plakar Control Plane URL, S3 bucket
names and more.

For following along with this demo I'd suggest using the versions defined
there, but if you're feeling adventurous you're more than welcome to use newer
versions.

## Create Cluster

On AWS with EKS it should take about 10-15 minutes to complete. It will create
a (2) node cluster using `t3.medium` instances:

```sh
./run cluster:create
```

### Sanity check

```sh
kubectl get nodes
```

```sh
NAME                                         STATUS   ROLES    AGE     VERSION
ip-10-100-1-30.us-east-2.compute.internal    Ready    <none>   4m59s   v1.37.1-eks-6e7947a
ip-10-100-2-126.us-east-2.compute.internal   Ready    <none>   5m      v1.37.1-eks-6e7947a
```

After ~20 seconds all of the pods should be ready:

```sh
kubectl get pods -A
```

```sh
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-7wxrz             2/2     Running   0          5m15s
kube-system   aws-node-fnk6j             2/2     Running   0          5m14s
kube-system   coredns-5cc8f84d8c-wghgj   1/1     Running   0          6m45s
kube-system   coredns-5cc8f84d8c-zdnw7   1/1     Running   0          6m45s
kube-system   kube-proxy-8fhbl           1/1     Running   0          5m14s
kube-system   kube-proxy-g28w5           1/1     Running   0          5m15s
```

### Install cluster dependencies

Given we're using EKS, there are a few things that will get installed and
configured for you:

- VPC / security groups for the cluster
- EKS Pod Identity Agent
- EBS and EFS CSI IAM roles
- EBS and EFS CSI driver
- Create an Elastic File System (EFS)
- Kubernetes Gateway API
- NGINX Gateway Fabric

This should take ~5 minutes to complete:

```sh
./run cluster:install-dependencies
```

## Deploy Core

This will set up a few things such as namespaces, a Kubernetes storageClass and
the gateway. Both the storageClass and gateway are configured in the `.env`
file.

This should take ~15 seconds to complete:

```sh
./run cluster:deploy-core
```

## Kubernetes Port Forward

Since we're not using a load balancer (to keep things simple), we'll want to
forward our apps' service port back to the host. Run this in a 2nd
terminal:

```sh
./run cluster:port-forward
```

The above will remain running in the foreground.

## Deploy Demo Apps

### Echo hello world

We'll start with a simple echo service to ensure the gateway is working. Even
though this is a demo app we'll still put it into a proper namespace just like
your workloads might be in production:

```sh
./run kubectl:apply 02-app/01-echo.yaml --namespace apps
```

After ~5 seconds you can visit the site and expect a greeting:

```sh
curl http://echo.localhost:8000
```

```sh
Hello from Kubernetes!
```

Check `02-app/01-echo.yaml` to see what this app is composed of. It's a
Deployment, Service and HTTPRoute.

### WordPress with MariaDB

This expands on the echo app by introducing a real web application that has a
connected database. It will let us test file uploads and includes using a
`PersistentVolumeClaim` for both WordPress and MariaDB.

It will also exercise running (2) replicas of WordPress with `ReadWriteMany` on
the `PersistentVolumeClaim` which will work over EFS.

```sh
./run kubectl:apply 02-app/02-wordpress.yaml --namespace apps
```

After about 2 minutes you can visit <http://wordpress.localhost:8000> in your
browser. It's normal to see the pods running for a bit while still seeing a
502 / 503 error until the back-end is ready.

#### Troubleshooting

If the site doesn't come up after 3 minutes, ensure everything is running:

```sh
kubectl --namespace apps get pods
```

```sh
NAME                                    READY   STATUS    RESTARTS   AGE
echo-588d88684c-5t4rm                   1/1     Running   0          76s
mariadb-deployment-776b8879c-wjznk      1/1     Running   0          56s
wordpress-deployment-585fc94476-867jc   1/1     Running   0          55s
wordpress-deployment-585fc94476-c5lxn   1/1     Running   0          55s
```

```sh
kubectl --namespace apps get httproute
```

```sh
NAME        HOSTNAMES                 AGE
echo        ["echo.localhost"]        45s
wordpress   ["wordpress.localhost"]   25s
```

```sh
kubectl --namespace apps get pvc
```

```sh
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mariadb-pvc     Bound    pvc-0be41c66-cd23-4a23-9a47-a29f6724f351   1Gi        RWO            gp3            <unset>                 30s
wordpress-pvc   Bound    pvc-9c7e70ca-fd00-4a40-8595-aeebcda74bc8   10Gi       RWX            efs-csi        <unset>                 29s
```

For this one, pay close attention to the conditions and check whether any
errors are reported:

```sh
kubectl describe pods wordpress --namespace apps
```

#### Setting up and testing WordPress

We'll want to quickly test file uploads and edit a blog post to make sure we
can save and restore everything through Plakar in a bit.

Setting up WordPress:

- Visit <http://wordpress.localhost:8000>
- Pick your language
- Set a site title
- Create your password and set your email address
- Click the install WordPress button

Log into the WordPress dashboard with your credentials:

- Edit the default post at <http://wordpress.localhost:8000/wp-admin/post.php?post=1&action=edit>
    - Adjust the title to say "Hello world from Kubernetes!"
- Add a picture of your choosing to the post
    - You can drag / drop it into the body of the post
- Save the post
- View the post at <http://wordpress.localhost:8000/?p=1>
    - Expect to see your edited title and photo

#### Verify Kubernetes volumes are working

Confirm the volumes are working by restarting the containers:

```sh
kubectl rollout restart deployment/mariadb-deployment --namespace apps
kubectl rollout restart deployment/wordpress-deployment --namespace apps
```

Wait about 30 seconds and then reload the page. You should see everything
load with the custom title and photo you uploaded.

At this point we've verified your cluster is in a good state. There are a few
apps running, including apps that use a volume through EFS. We're ready to
back up and restore the data using Plakar.

## Plakar Kubernetes Operator

[This
operator](https://www.plakar.io/docs/control-plane/infrastructure-as-code/kubernetes-operator/installation/)
lets you back up and restore workloads running in Kubernetes by letting you
define specific types of Kubernetes configs. Behind the scenes it makes API
calls to your Plakar Control Plane.

### IMPORTANT: Update the `.env` file

Before continuing, please make sure the `.env` file is configured with the
values you want.

Specifically these values, there are comments for each one in the `.env` file:

- `AWS_DEFAULT_REGION`
- `AWS_S3_BUCKET_NAME`
- `PLAKAR_CONTROL_PLANE_EC2_IAM_ROLE_NAME`
- `PLAKAR_KUBERNETES_OPERATOR_PCP_URL`
    - The URL of your Plakar Control Plane (i.e. `https://pcp.example.com/` or `http://42.42.42.42/` if you're testing it directly on an instance without a domain)

### Configuring cluster access

First we'll want to create a Kubernetes ServiceAccount with a bearer token so
PCP can communicate with your EKS cluster. We'll also set up a few RBAC
permissions so this account can only do the bare minimum to back up and restore
PVCs.

All of the above is handled by this command which should finish in a few
seconds:

```sh
./run cluster:configure-plakar-access
```

### Installing the operator in your cluster

This should finish in ~20 seconds:

```sh
./run cluster:install-plakar-operator
```

### Generating an API key

Next up we need to generate an API key in your Plakar Control Plane so the
operator can make API calls up to your control plane. This is the first part of
the demo where a control plane is required.

- Visit your control plane's dashboard
- Click "Organizations" on the left
- Click "Add user"
- Choose "Application" as the type of user
- Name it anything you'd like such as "Kubernetes Operator"

Now we need to create an API key for the user we just created above:

- For the description, use anything you'd like such as the name of your cluster
- Copy your API key somewhere safe (password manager, etc.) as it won't be shown again

### Permissions for this user

If you want this application user to be able to self-manage its own resources
such as your Plakar Control Plane inventory, it must have permissions to do so.

You can follow these steps to do that:

- In the organization page, click the "Permissions" tab up top
- Click "Grant permission"
- Choose "Applications" as the subject
- Select the application user you just created
- Select the "Operator" role
- Click "Grant permission"

### Connect Kubernetes Operator to your control plane

We'll add a secret to our Kubernetes cluster with the API key, then in the
future, configs that need it will read from this secret instead of hard coding
it.

*The command below is formatted to start with a space so it doesn't get saved
to your shell's history in most cases. Double check you have no leading or
trailing spaces in your key:*

```sh
 kubectl create secret generic plakar-credentials \
   --namespace plakar-operator-system \
   --from-literal=apikey="pcp_ak_TODO_PASTE_YOUR_API_KEY_HERE"
```

If you ever need to re-roll your API key in the future you can generate a new
one and then run:

```sh
 kubectl create secret generic plakar-credentials \
   --namespace plakar-operator-system \
   --from-literal=apikey="pcp_ak_TODO_PASTE_YOUR_API_KEY_HERE" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Manually install a few integrations

Since we only gave the app user "operator" permissions, the operator can't
install integrations. This is something that will be improved on over time.

For now, in the PCP UI (Plakar Control Plane UI) visit the "Integrations"
section and install both the "S3" and "Kubernetes" integrations. You can click
the install button for each one.

These are one-time operations you need to do.

## Operator Configuration

There's a number of things to configure for the operator. Each one is numbered
and you can find them in `03-plakar-operator/XX-example.yaml` if you want to
see what each one does.

The `00-rbac.yaml` config was already applied when we set up access but the
others will be applied one at a time manually by us so we can walk through each
one.

### 1. Inventory

By default we'll let the operator create and manage a new Plakar inventory for
you. If you have an existing inventory please check out the `.env` file for
`PLAKAR_KUBERNETES_OPERATOR_INVENTORY_UUID` and put your UUID in. You can
follow the comments in the `.env` file.

I'd suggest letting the operator manage a new inventory:

```sh
./run kubectl:apply 03-plakar-operator/01-plakar.yaml --namespace plakar-operator-system
```

We'll be adding it to the `plakar-operator-system` namespace which will contain
a few Plakar related resources.

#### Confirm it works

The important takeaway is the status says it's successfully loaded and
available:

```sh
kubectl --namespace plakar-operator-system describe plakar pcp
```

```sh
[...]
Status:
  Conditions:
    Message:               Configuration successfully loaded
    Reason:                ConfigurationLoaded
    Status:                True
    Type:                  Available
```

### 2. Store

Next up we'll configure our store. This is where your backups will be managed
by Plakar. In our case it will be an S3 bucket. This uses the "S3" integration
we installed earlier.

Before we create it, we need to create a few secrets that will be referenced
in the store's Kubernetes config.

These are being done by hand so you have an awareness of them.

The commands below contain secrets and have a leading space so they don't get
saved to your shell's history.

Also, they will be added to the `apps` namespace which is where we'll apply the
rest of the Plakar resources. It's also where your applications are running
(WordPress in this case).

#### AWS access keys

These are associated with your Plakar IAM user:

```sh
 kubectl create secret generic aws-credentials \
   --namespace apps \
   --from-literal=accessKey="TODO_PASTE_YOUR_AWS_ACCESS_KEY" \
   --from-literal=secretAccessKey="TODO_PASTE_YOUR_AWS_SECRET_KEY"
```

#### Kloset passphrase

This is managed by Plakar and it's used to encrypt and decrypt your store:

```sh
 kubectl create secret generic kloset \
   --namespace apps \
   --from-literal=passphrase="TODO_PROVIDE_A_GOOD_PASSPHRASE"
```

#### Applying it

```sh
./run kubectl:apply 03-plakar-operator/02-store.yaml --namespace apps
```

#### Initialize the store

If you were to run `kubectl --namespace apps describe store aws-s3` it will
fail with a "The specified key does not exist" error. This is temporary
because the operator doesn't initialize the store yet (it will in the future).

We have to do it manually in the Plakar Control Plane UI:

- Go to "Inventories" in the sidebar
- Click the "Stores" tab
- Click the "aws-s3" store
- Click the "Initialize store" button

Once that completes successfully we need to trigger a change so that the store
reconciles on the Kubernetes side of things, you can do that by temporarily
changing one of its properties, such as:

```sh
kubectl patch store aws-s3 -n apps --type=merge --patch '{"spec":{"fields":{"root":{"value":"/"}}}}'
kubectl patch store aws-s3 -n apps --type=merge --patch '{"spec":{"fields":{"root":null}}}'
```

Once that happens, the pod will get recreated and it should work.

#### Confirm it works

You'll want to see that the connector is reachable:

```sh
kubectl --namespace apps describe store aws-s3
```

```sh
[...]
Status:
  Conditions:
    Message:               Underlying connector updated at plakar control-plane
    Reason:                Updated
    Status:                True
    Type:                  Available
    Message:               connector is reachable
    Reason:                ConnectionVerified
    Status:                True
    Type:                  Tested
```

You'll see an assortment of different errors depending on what went wrong.
For example you may see a 409 error if your AWS credentials are incorrect.

### 3. Source

Next up we need to create a source, i.e. what we're backing up. In this
case it will be the PVC for both WordPress and MariaDB. This will use the
Kubernetes integration that was installed before.

#### Applying it

```sh
./run kubectl:apply 03-plakar-operator/03-source.yaml --namespace apps
```

It's important that the `spec.endpoint` is a full path of your namespace
("apps") and the resource being backed up. Also, it references a `kubeconfig`
secret that was created when `cluster:configure-plakar-access` was run. This
contains a long-lived ServiceAccount user token to provide cluster access.

#### Confirm it works

You'll want to see that the connector is reachable:

```sh
kubectl --namespace apps describe source wordpress-pvc
kubectl --namespace apps describe source mariadb-pvc
```

```sh
Status:
  Conditions:
    Message:               Underlying connector created at plakar control-plane
    Reason:                Created
    Status:                True
    Type:                  Available
    Message:               connector is reachable
    Reason:                ConnectionVerified
    Status:                True
    Type:                  Tested
```

#### Getting your source IDs for the `.env` file

There are a few IDs to add to the `.env` file, once your sources are created
these IDs won't change.

- In the PCP UI, go to "Inventories" in the sidebar
- Expand the `mariadb-pvc` item (*not `mariadb-restore`*)
- Under the "Apps" tab, there is a Kubernetes icon with an ID next to it
- Copy the ID using the little icon in the UI
- Add this ID to `PLAKAR_SOURCE_ID_MARIADB` in the `.env` file
- Repeat the same thing for `wordpress-pvc`

We'll be using this later on when we do a restore.

### 4. Destination

This is similar to the source. In this case it tells Plakar where to write data
to when restoring. It's important to understand that the destination itself
doesn't contain the data. It's a registered connection and a target location
for restored data.

#### Applying it

```sh
./run kubectl:apply 03-plakar-operator/04-destination.yaml --namespace apps
```

In this case, the `spec.endpoint` ends up being the same as the source. The
same PVC is used for both. That is a preference for this demo, you can use
different PVCs if you want. Just make sure any other PVCs are created ahead of
time because the RBAC permissions of the ServiceAccount cannot create PVCs as
is.

#### Confirm it works

You'll want to see that the connector is reachable:

```sh
kubectl --namespace apps describe destination wordpress-restore
kubectl --namespace apps describe destination mariadb-restore
```

```sh
Status:
  Conditions:
    Message:               Underlying connector created at plakar control-plane
    Reason:                Created
    Status:                True
    Type:                  Available
    Message:               connector is reachable
    Reason:                ConnectionVerified
    Status:                True
    Type:                  Tested
```

### 5. Manual backup

Before we start scheduling backups, we can test our integrations and store by
doing a 1-off backup.

#### Applying it

```sh
./run kubectl:apply 03-plakar-operator/05-backup-now.yaml --namespace apps
```

If you want to create more than 1 backup you can delete + apply it.

#### Confirm it works

In the PCP UI you can go to "Scheduling" in the sidebar, then pick the
"Kubernetes Operator" up top in the dropdown to pick the user. Then click
the "Job history" tab.

Once there you'll see backup results. Ideally it should be successful once it
finishes. If not you can click the details of the job and look at the logs.

What's nice is, in that same job details page you can click the "Browse restore
point" link and see the files that have been backed up. For example with
WordPress you can go to `wp-content/uploads` and find the photo you uploaded to
verify it's there.

### 5. Schedule backup

Now we're ready to have our backups get created on a regular basis, such as
once per day.

#### Applying it

```sh
./run kubectl:apply 03-plakar-operator/05-schedule-backup.yaml --namespace apps
```

There's `spec.periodicity` for the interval on when it'll run, it's set to 1
day in seconds. There's also a `historyLimit` set to 7 which is not the number
of retained backups. It's the number of Kubernetes tasks to keep around.

In the future the operator will be modified to support more flexible times
and being able to set a max retention policy. The Terraform provider allows
this already.

#### Confirm it works

You can use the same method as doing a manual backup. If you don't want to wait
a day you can set the `periodicity` to 5 minutes or so and see if it runs
successfully, then change it back afterwards to once per day.

### 6. Restore

Life wouldn't be complete unless we also tested our backup to make sure we can
restore it.

#### Create a new post

We'll add a 2nd post and eventually restore our old backup with an expectation
that we only see 1 post.

- Visit <http://wordpress.localhost:8000/wp-admin/post-new.php> to make a new post
- Create a new post with any title
- Upload a photo of your choosing as the post body
- Save the post

#### Spin things down

Since the same PVC is used for the source and destination, stop both MariaDB
and WordPress before restoring. This avoids race conditions and makes sure
nothing gets lost between the restore and whatever someone might be doing in
WordPress.

```sh
kubectl --namespace apps scale deployment wordpress-deployment --replicas=0
kubectl --namespace apps scale deployment mariadb-deployment --replicas=0
```

#### Applying it

Before applying it, make sure your source IDs are added to your `.env` file.
Revisit the source section to see how to do that. That's number 3 (we're on 6
now).

```sh
./run kubectl:apply 03-plakar-operator/06-restore.yaml --namespace apps
```

There are a few important callouts here: `spec.filters.dataset` combined with
`latest` lets us restore from the latest snapshot. If we didn't supply the
dataset then it would pick the literal last snapshot of the store and that
would be a problem. We need to differentiate MariaDB vs WordPress which is
done by the source ID.

Alternatively, if you want to pick a specific snapshot ID to restore from you
can comment out the `spec.filters` and uncomment `spec.snapshotID` and then
supply a specific snapshot ID to restore from. You can find it in the PCP UI
near the same spot where you copied the source ID: find the snapshot in the
list below it, click the "Browse" menu item and copy the ID from there.

#### Confirm it works

```sh
kubectl --namespace apps describe restore wordpress-restore
kubectl --namespace apps describe restore mariadb-restore
```

```sh
Status:
  Conditions:
    Message:               restore completed successfully
    Reason:                Completed
    Status:                True
    Type:                  Available
```

Like backing up, if it fails you can view the details of the job in the PCP UI
and see what went wrong.

#### Spin things up

Now that we've restored, we can run our workloads again, which takes ~30
seconds:

```sh
kubectl --namespace apps scale deployment wordpress-deployment --replicas=2
kubectl --namespace apps scale deployment mariadb-deployment --replicas=1
```

If you visit <http://wordpress.localhost:8000> you should see only 1 blog post.
We had 2 previously but since we restored from a backup that only had 1, only 1
should exist. You can also visit
<http://wordpress.localhost:8000/wp-admin/upload.php> and ensure you only see 1
uploaded image.

## Operator Details

Besides [the API
documentation](https://www.plakar.io/docs/control-plane/references/kubernetes-operator/)
you can always use `kubectl` commands to learn more about the custom CRDs
introduced by this operator. I sometimes find this faster than looking up the
specs in the docs.

#### Basic information

```sh
kubectl explain plakar.spec --recursive
kubectl explain store.spec --recursive
kubectl explain source.spec --recursive
kubectl explain destination.spec --recursive
kubectl explain backup.spec --recursive
kubectl explain schedulebackup.spec --recursive
kubectl explain restore.spec --recursive
```

#### Extended information

```sh
kubectl get crd plakars.task.plakar.io --output yaml
kubectl get crd stores.connector.plakar.io --output yaml
kubectl get crd sources.connector.plakar.io --output yaml
kubectl get crd destinations.connector.plakar.io --output yaml
kubectl get crd backups.task.plakar.io --output yaml
kubectl get crd schedulebackups.task.plakar.io --output yaml
kubectl get crd restores.task.plakar.io --output yaml
```
