# Google Terraform Quickstart template

使用此模板轻松创建新的 Git 存储库，以管理 Jenkins X 云基础设施需求。

我们建议使用 Terraform 来管理运行 Jenkins X 所需的基础设施。可能需要创建一些云资源，例如：

- Kubernetes cluster
- Storage buckets 用于长期存储日志
- IAM Bindings 用于管理使用云资源的应用程序权限

Jenkins X 喜欢使用 GitOps 来管理基础设施和集群资源的生命周期。这需要两个 Git 存储库来实现：
- **Infrastructure git repository**: 基础设施资源将由 Terraform 管理，并保持资源同步
- **Cluster git repository**: Kubernetes 特定的集群资源将由 Jenkins X 管理，并保持资源同步。

# 先决条件

- A Git organisation that will be used to create the GitOps repositories used for Jenkins X below.
  e.g. https://github.com/organizations/plan.
- Create a git bot user (different from your own personal user)
  e.g. https://github.com/join
  and generate a a personal access token, this will be used by Jenkins X to interact with git repositories.
  e.g. https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,write:repo_hook,delete_repo,admin:repo_hook

- __This bot user needs to have write permission to write to any git repository used by Jenkins X.  This can be done by adding the bot user to the git organisation level or individual repositories as a collaborator__
  Add the new `bot` user to your Git Organisation, for now give it Owner permissions, we will reduce this to member permissions soon.
- Check and install latest `terraform` CLI - [see here](https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/aws-get-started)
- Check and install latest `jx` CLI - [see here](https://jenkins-x.io/v3/admin/setup/jx3/)
- Check and install latest `gcloud` CLI - [see here](https://cloud.google.com/sdk/docs/install)
- Setup local `gcloud` auth so that Terraform can work with GCP
```bash
gcloud auth application-default login
```
- Google cloud container services has to be enabled
```bash
gcloud services enable container.googleapis.com
```

# Git repositories

We use 2 git repositories:

* **Infrastructure** 用于 Terraform 配置的 Git 存储库，用于设置/升级/修改您的云基础设施（Kubernetes 集群、IAM 账户、IAM 角色、存储桶等）
  
* **Cluster** 包含 `helmfile.yaml` 文件的 Git 存储库，用于定义要在您的集群中部署的 Helm charts

我们使用独立的 git 仓库，因为基础架构往往很少更改；而集群 git 仓库却经常更改（每次添加新的快速启动、导入项目、发布项目等）。

通常由不同的团队负责基础架构；或者您可以使用 Terraform Cloud 等工具来处理基础架构的变更，并对基础架构的变更进行比推广应用更严格的审查。

# Getting started

__Note:请记住在您的 Git 组织而不是您的个人 Git 账户中创建下面的 Git 仓库，否则会导致 ChatOps 和自动注册 webhooks___ 的问题。

1. Create an **Infrastructure** git repo from this GitHub Template https://github.com/jx3-gitops-repositories/jx3-terraform-gke/generate.  If you are following from the Jenkins X website then this initial step may have already happened.

    __Note:__ Ensure **Owner** is the name of the Git Organisation that will hold the GitOps repositories used for Jenkins X.

2. Create a **Cluster** git repository; choosing your desired secrets store, either Google Secret Manager or Vault:
    - __Google Secret Manager__: https://github.com/jx3-gitops-repositories/jx3-gke-gsm/generate 
    __Note:__ 如果选择 Google Secret Manager，则必须在您的账户上激活计费！
    

    - __Vault__: https://github.com/jx3-gitops-repositories/jx3-gke-vault/generate
    
    __Note:__ Ensure **Owner** is the name of the Git Organisation that will hold the GitOps repositories used for Jenkins X.

3. You need to configure the git URL of your **Cluster** git repository (which contains `helmfile.yaml`) into the **Infrastructure** git repository (which contains `main.tf`). 

So from inside a git clone of the **Infrastructure** git repository (which already has the files `main.tf` and `values.auto.tfvars` inside) you need to link to the other **Cluster** repository (which contains `helmfile.yaml`) by committing the required terraform values from below to your `values.auto.tfvars`, e.g.

```sh
cat <<EOF >> values.auto.tfvars    
jx_git_url = "https://github.com/$git_owner_from_cluster_template_above/$git_repo_from_cluster_template_above"
gcp_project = "my-gcp-project"
EOF
```
If using Google Secret Manager (not Vault) cluster template from above enable it for Terraform using:
```sh
cat <<EOF >> values.auto.tfvars 
gsm = true
EOF
```

The contents of your `values.auto.tfvars` file should look something like this (the last line will be omitted if not using gsm)....

```terraform
resource_labels = { "provider" : "jx" }
jx_git_url = "https://github.com/myowner/myname-cluster"
gcp_project = "my-gcp-project"
gsm = true
```

4. commit and push any changes to your **Infrastructure** git repository:

```sh
git commit -a -m "fix: configure cluster repository and project"
git push
```

5. Now define 2 environment variables to pass the bot user and token into Terraform:

```sh
export TF_VAR_jx_bot_username=my-bot-username
export TF_VAR_jx_bot_token=my-bot-token
```

6. Now, initialise, plan and apply Terraform:

```sh
terraform init
```

```sh
terraform plan
```

```sh
terraform apply
```

Connect to the cluster
```
$(terraform output connect)
```
Tail the Jenkins X installation logs
```
$(terraform output follow_install_logs)
```
Once finished you can now move into the Jenkins X Developer namespace

```sh
jx ns jx
```

and create or import your applications

```sh
jx project
```

## Terraform Inputs

For the full list of terraform inputs [see the documentation for jenkins-x/terraform-google-jx](https://github.com/jenkins-x/terraform-google-jx#inputs)

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_apex_domain"></a> [apex\_domain](#input\_apex\_domain) | The apex / parent domain to be allocated to the cluster | `string` | `""` | no |
| <a name="input_apex_domain_gcp_project"></a> [apex\_domain\_gcp\_project](#input\_apex\_domain\_gcp\_project) | The GCP project the parent domain is managed by, used to write recordsets for a subdomain if set.  Defaults to current project. | `string` | `""` | no |
| <a name="input_apex_domain_integration_enabled"></a> [apex\_domain\_integration\_enabled](#input\_apex\_domain\_integration\_enabled) | Add recordsets from a subdomain to a parent / apex domain | `bool` | `true` | no |
| <a name="input_artifact_description"></a> [artifact\_description](#input\_artifact\_description) | artifact registry repository Description | `string` | `"jenkins-x Docker Repository"` | no |
| <a name="input_artifact_enable"></a> [artifact\_enable](#input\_artifact\_enable) | Create artifact registry repository | `bool` | `true` | no |
| <a name="input_artifact_location"></a> [artifact\_location](#input\_artifact\_location) | artifact registry repository Location | `string` | `"us-central1"` | no |
| <a name="input_artifact_repository_id"></a> [artifact\_repository\_id](#input\_artifact\_repository\_id) | artifact registry repository Name, Defaul Cluster Name | `string` | `""` | no |
| <a name="input_autoscaler_location_policy"></a> [autoscaler\_location\_policy](#input\_autoscaler\_location\_policy) | location policy for primary node pool | `string` | `"ANY"` | no |
| <a name="input_autoscaler_max_node_count"></a> [autoscaler\_max\_node\_count](#input\_autoscaler\_max\_node\_count) | Maximum number of cluster nodes | `number` | `5` | no |
| <a name="input_autoscaler_min_node_count"></a> [autoscaler\_min\_node\_count](#input\_autoscaler\_min\_node\_count) | Minimum number of cluster nodes | `number` | `3` | no |
| <a name="input_cluster_location"></a> [cluster\_location](#input\_cluster\_location) | The location (region or zone) in which the cluster master will be created. If you specify a zone (such as us-central1-a), the cluster will be a zonal cluster with a single cluster master. If you specify a region (such as us-west1), the cluster will be a regional cluster with multiple masters spread across zones in the region | `string` | `"us-central1-a"` | no |
| <a name="input_cluster_name"></a> [cluster\_name](#input\_cluster\_name) | Name of the Kubernetes cluster to create | `string` | `""` | no |
| <a name="input_force_destroy"></a> [force\_destroy](#input\_force\_destroy) | Flag to determine whether storage buckets get forcefully destroyed | `bool` | `false` | no |
| <a name="input_gcp_project"></a> [gcp\_project](#input\_gcp\_project) | The name of the GCP project to use | `string` | n/a | yes |
| <a name="input_gsm"></a> [gsm](#input\_gsm) | Enables Google Secrets Manager, not available with JX2 | `bool` | `false` | no |
| <a name="input_initial_cluster_node_count"></a> [initial\_cluster\_node\_count](#input\_initial\_cluster\_node\_count) | initial number of cluster nodes | `number` | `3` | no |
| <a name="input_initial_primary_node_pool_node_count"></a> [initial\_primary\_node\_pool\_node\_count](#input\_initial\_primary\_node\_pool\_node\_count) | initial number of pool nodes | `number` | `1` | no |
| <a name="input_jx_bot_token"></a> [jx\_bot\_token](#input\_jx\_bot\_token) | Bot token used to interact with the Jenkins X cluster git repository | `string` | n/a | yes |
| <a name="input_jx_bot_username"></a> [jx\_bot\_username](#input\_jx\_bot\_username) | Bot username used to interact with the Jenkins X cluster git repository | `string` | n/a | yes |
| <a name="input_jx_git_url"></a> [jx\_git\_url](#input\_jx\_git\_url) | URL for the Jenins X cluster git repository | `string` | n/a | yes |
| <a name="input_kuberhealthy"></a> [kuberhealthy](#input\_kuberhealthy) | Enables Kuberhealthy helm installation | `bool` | `false` | no |
| <a name="input_lets_encrypt_production"></a> [lets\_encrypt\_production](#input\_lets\_encrypt\_production) | Flag to determine wether or not to use the Let's Encrypt production server. | `bool` | `true` | no |
| <a name="input_master_authorized_networks"></a> [master\_authorized\_networks](#input\_master\_authorized\_networks) | List of master authorized networks. If none are provided, disallow external access (except the cluster node IPs, which GKE automatically allowlists). | `list(object({ cidr_block = string, display_name = string }))` | <pre>[<br>  {<br>    "cidr_block": "0.0.0.0/0",<br>    "display_name": "any"<br>  }<br>]</pre> | no |
| <a name="input_node_disk_size"></a> [node\_disk\_size](#input\_node\_disk\_size) | Node disk size in GB | `string` | `"100"` | no |
| <a name="input_node_disk_type"></a> [node\_disk\_type](#input\_node\_disk\_type) | Node disk type, either pd-standard or pd-ssd | `string` | `"pd-standard"` | no |
| <a name="input_node_machine_type"></a> [node\_machine\_type](#input\_node\_machine\_type) | Node type for the Kubernetes cluster | `string` | `"n1-standard-2"` | no |
| <a name="input_node_preemptible"></a> [node\_preemptible](#input\_node\_preemptible) | Use preemptible nodes | `bool` | `false` | no |
| <a name="input_node_spot"></a> [node\_spot](#input\_node\_spot) | Use spot nodes | `bool` | `false` | no |
| <a name="input_resource_labels"></a> [resource\_labels](#input\_resource\_labels) | Set of labels to be applied to the cluster | `map(string)` | `{}` | no |
| <a name="input_subdomain"></a> [subdomain](#input\_subdomain) | Optional sub domain for the installation | `string` | `""` | no |
| <a name="input_tls_email"></a> [tls\_email](#input\_tls\_email) | Email used by Let's Encrypt. Required for TLS when parent\_domain is specified | `string` | `""` | no |

# Cleanup

To remove any cloud resources created here run:
```sh
terraform destroy
```

# Contributing

When adding new variables please regenerate the markdown table 
```sh
terraform-docs markdown table .
```
and replace the Inputs section above

## Formatting

When developing please remember to format codebase before raising a pull request
```sh
terraform fmt -check -diff -recursive
```
