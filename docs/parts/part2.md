<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 2: Kubernetes Deployment and Configuration

The following steps require that you be familiar with HashiCorp Terraform modules. For more info, please read [Terraform's documentation](https://developer.hashicorp.com/terraform/tutorials/modules/module). The steps below will guide you to deploy Kubernetes with a Terraform module in an automated fashion.

## Steps

### 1. Clone the terraform-equinix-kubernetes-cluster repo

For this workshop, we will clone the repository [`terraform-equinix-kubernetes-cluster`](https://github.com/equinix-labs/terraform-equinix-kubernetes-cluster), to provision a kubernetes cluster. This repository contains a Terraform module that you will run to provision the cluster.

```shell
$ git clone https://github.com/equinix-labs/terraform-equinix-kubernetes-cluster.git
$ cd terraform-equinix-kubernetes-cluster
```

### 2. Customize the Terraform Deployment

Now that we have cloned the kubernetes repo, we will need to customize our deployment. Let's first create a subfolder for our custom deployment under the `examples` folder in the repo and call it `my-deployment`.

```shell
$ cd examples && mkdir my-deployment && cd my-deployment
```

We need a Container Network Interface (CNI) plug-in in order to run applications on the Kubernetes cluster. Luckily, there already is a deployment of the [`flannel` CNI plug-in](https://github.com/dcallao/terraform-equinix-kubernetes-cluster/tree/main/examples/cluster-with-cni) under the `examples` folder. Let's copy the contents under that folder to customize our deployment.

```shell
$ cp ../cluster-with-cni/* .
```

Now we need to assign the variables used in the `module` block located in the file `main.tf` from the previous step with attributes it needs to run. There are multiple ways to accomplish that. For demonstration, we'll populate the `terraform.tfvars` file with the variable assignments. For more info on `terraform.tfvars`, see [`Assign values with a file`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables#assign-values-with-a-file) documentation. For other options, see [`Customize Terraform configuration with variables`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables)

Remember to assign the `metal_auth_token` and the `metal_project_id` variables with the values obtained from Part 1 of the workshop.

```shell
$ mv terraform.tfvars.example terraform.tfvars
```

```terraform
# This must be a user API token
metal_auth_token        = "pUqwuRtmQ3cMZBKodr1arir5GejbFNsp"
metal_project_id        = "16a060ea-9de3-4cd1-3601-49d38495d426"
# Optional, where you want to store your SSH key
kube_vip_version        = "v0.6.2"
kubernetes_version      = "v1.28.1"
metal_metro             = "da"
# Optional, choose a specific flannel version, default is v0.24.2
# flannel_version      = "v0.24.2"
# Optional, name of a specific cloud provider. Possible values: external, YOUR_CLOUD_PROVIDER, "" (empty for internal cloud controller)
cloud_provider_external = false
```

> **_Note:_** You may build custom documentation in README.md but it does not necessarily need to be populated in order to provision infrastructure.

### 3. Install the Kubernetes Command Line Tool: kubectl

There are many [tools](https://kubernetes.io/docs/tasks/tools/) available to manage a Kubernetes cluster. For demonstration, we will focus on `kubectl` as our Kubernetes CLI manager of choice.

To install `kubectl` on MacOS, please follow [these instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/).

We are ready to run Terraform to provision the kubernetes cluster!

### 4. Provision Kubernetes

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
  "cloud_init_done" = "3049598830591298943"
  "kubeconfig_ready" = "6387064462670964962"
  "kubeip_vip" = "86.109.9.237"
}
```

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
    server: https://86.109.9.237:6443
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

Note that the cluster server IP `86.109.9.237` matches the `kubeip_vip` output posted in the outputs of the terraform run completed in Step 3.

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* How many ways can a Kubernetes cluster be managed?
