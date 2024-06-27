<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 2: Kubernetes Addons Deployment and Configuration

The following steps require that you be familiar with HashiCorp Terraform modules. For more info, please read [Terraform's documentation](https://developer.hashicorp.com/terraform/tutorials/modules/module). The steps below will guide you to deploy a Kubernetes cluster with Kubernetes addons with Terraform modules in an automated fashion.

## Steps

### 1. Clone the terraform-equinix-kubernetes-cluster repo

For this workshop, we will clone the repository [`terraform-equinix-kubernetes-cluster`](https://github.com/equinix-labs/terraform-equinix-kubernetes-cluster), to provision a kubernetes cluster. This repository contains a Terraform module that you will run to provision the Kubernetes cluster.

```shell
$ git clone https://github.com/equinix-labs/terraform-equinix-kubernetes-cluster.git
$ cd terraform-equinix-kubernetes-cluster
```

### 2. Customize the Deployment with Kubernetes Addons

Now that we have cloned the kubernetes repo, we will need to customize our deployment. Let's first create a subfolder for our custom deployment under the `examples` folder in the repo and call it `my-deployment`.

```shell
$ cd examples && mkdir my-deployment && cd my-deployment
```

We are going to take a look into the [terraform-equinix-kubernetes-addons](https://github.com/equinix-labs/terraform-equinix-kubernetes-addons/tree/main) project and choose which kubernetes addon we would like to deploy into the cluster. By definition, the kubernetes addons project defines a set of Terraform modules. Each module represents a feature or functionality (i.e. load balancing) that users can choose to add into a kubernetes custer deployment. Let's look at those addons closely:

```shell
$ git clone https://github.com/equinix-labs/terraform-equinix-kubernetes-addons.git
$ cd terraform-equinix-kubernetes-addons
$ ls -al modules
```

Upon checking the list of addons available, we are going to add the Cloud Provider Equinix Metal (CPEM) addon into our kubernetes deployment.

Cloud Controller Manager (CPEM) links the Kubernetes cluster to Metal's APIs. Luckily, there already is a deployment of [CPEM](https://github.com/dcallao/terraform-equinix-kubernetes-cluster/tree/main/examples/cpem-addon) under the `examples` folder. Let's copy the contents under that folder to customize our deployment.

```shell
$ cp ../cpem-addon/* .
```

Now we need to add the addons as a new `module` in the `main.tf` file. Let's add the new block as such:

```
module "kubernetes_addons" {
  source  = "equinix-labs/kubernetes-addons/equinix"
  version = "0.4.0"

  ssh_user        = replace("root", module.tfk8s.kubeconfig_ready, "")
  ssh_private_key = var.ssh_private_key_path == "" ? join("\n", [chomp(module.tfk8s.ssh_key_pair[0].private_key_openssh), ""]) : chomp(file(var.ssh_private_key_path))
  ssh_host        = module.tfk8s.kubeapi_vip

  # Wait to run addons until the cluster is ready for them
  kubeconfig_remote_path = "/etc/kubernetes/admin.conf"

  # TODO: These aren't used for CPEM addon but we have to provide them
  equinix_metro         = var.metal_metro
  equinix_project       = var.metal_project_id
  kubeconfig_local_path = ""

  enable_cloud_provider_equinix_metal = true
  enable_metallb                      = false

  cloud_provider_equinix_metal_config = {
    version = var.cpem_version
    secret = {
      projectID    = var.metal_project_id
      apiKey       = var.metal_auth_token
      loadbalancer = "kube-vip://"
    }
  }
}
```

In the block above, We have enabled CPEM with `enable_cloud_provider_equinix_metal=true` statement:

```
  enable_cloud_provider_equinix_metal = true
  enable_metallb                      = false
```

We have also set additional variables `var.ssh_private_key_path` and `var.cpem_version` required by the CPEM addon to function. Let's define those in the `variables.tf` file as such:

```
# CPEM variables
variable "ssh_private_key_path" {
  description = "Path of the private key used to SSH into cluster nodes"
  sensitive   = true
  type        = string
  default     = ""
}

variable "cpem_version" {
  description = "Version of the CPEM"
  type        = string
  default     = "v3.8.0"
}
```

Now we need to assign the variables used in the `module` block located in the file `main.tf` from the previous step with attributes it needs to run. There are multiple ways to accomplish that. For demonstration, we'll populate the `terraform.tfvars` file with the variable assignments. For more info on `terraform.tfvars`, see [`Assign values with a file`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables#assign-values-with-a-file) documentation. For other options, see [`Customize Terraform configuration with variables`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables)

```shell
$ mv terraform.tfvars.example terraform.tfvars
```

Assign the `metal_auth_token` and the `metal_project_id` variables with the values obtained from Part 1 of the workshop. Keep the `metal_metro` variable and remove the remaining variable assignments.

```terraform
metal_auth_token        = "pUqwuRtmQ3cMZBKodr1arir5GejbFNsp"
metal_project_id        = "16a060ea-9de3-4cd1-3601-49d38495d426"
metal_metro             = "da"
```

> **_Note:_** You may build custom documentation in README.md but it does not necessarily need to be populated in order to provision infrastructure.

### 3. Provision Kubernetes

Now that we have finished building the terraform plan, we need to apply it. Let's take the same steps demonstrated in [`Part 3: Apply a Terraform Plan`](https://equinix-labs.github.io/terraform-on-equinix-workshop/parts/apply-plan/) of the the `Terraform on Equinix` workshop.

```shell
$ terraform init --upgrade
$ terraform plan
$ terraform apply -auto-approve
```

Once the terraform apply execution ends, you will see something like this:

```terraform
Apply complete! Resources: 17 added, 0 changed, 0 destroyed.

Outputs:

tfk8s_outputs = {
  "cloud_init_done" = "4420275658402557652"
  "kubeconfig_ready" = "990381990483175064"
  "kubeip_vip" = "145.40.90.140"
}
```

### 4. Install the Kubernetes Command Line Tool: kubectl

There are many [tools](https://kubernetes.io/docs/tasks/tools/) available to manage a Kubernetes cluster. For demonstration, we will focus on `kubectl` as our Kubernetes CLI manager of choice.

To install `kubectl` on MacOS, please follow [these instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/).

We are ready to run Terraform to provision the kubernetes cluster!

### 5. Configure the Kubernetes Command Line Tool: kubectl

`kubectl` sets the default config file path to `$HOME/.kube/config`. We will populate this config file with the contents of the file `kubeconfig.admin.yaml` file generate at the root of the workspace folder `my-deployment` from the previous Step 3.

```shell
$ cat kubeconfig.admin.yaml > $HOME/.kube/config
```

Alternatively, you could reference the generated config file `kubeconfig.admin.yaml` directly to manage the cluster. For instance:

```shell
$ kubectl --kubeconfig kubeconfig.admin.yaml get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   68m
```

Now that you have configured `kubectl`, let's verify its configuration:

```shell
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://145.40.90.140:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

Note that the cluster server IP `145.40.90.140` matches the `kubeip_vip` output posted in the outputs of the terraform run completed in Step 3.

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* How many ways can a Kubernetes cluster be managed?
