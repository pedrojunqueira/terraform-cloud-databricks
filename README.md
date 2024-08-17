# terraform-cloud-databricks


First what is terraform cloud?

It is a hosted service developed by HashiCorp that provides a collaborative workspace for teams to use Terraform, an open-source Infrastructure as Code (IaC) software tool.
In other words it helps to manage your terraform state files and change history in the cloud.

What is GitHub actions? 

 a continuous integration and continuous delivery (CI/CD) platform that allows you to automate your build, test, and deployment pipeline.

 I have done some videos using it you can watch in the channel.

 Ok for this tutorial you would need.

 VS Code
 Git installed
 A GitHub Account (free)
 A Terraform Account (free)
 A Azure Databricks workspace where you have admin rights

 So lets get started

 Steps

 1. Create a GitHub Repository call it terraform-cloud-databricks. .gitignore terraform
  https://github.com/pedrojunqueira?tab=repositories
 2. Clone in your local machine and create a feature branch
 3. Open VS Code (code .) Create a terraform main.tf file
 4. Copy and paste this code from databricks terraform provider. I will explain what it does.
    https://registry.terraform.io/providers/databricks/databricks/latest/docs
    https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/cluster
    Reduce the number of workers

```yml
terraform {

    required_version = ">= 1.9"
    required_providers {
    databricks = {
      source = "databricks/databricks"
    }
  }
}

variable "databricks_host" {
  type = string

}


variable "token" {
  type = string

}

provider "databricks" {
  host  = var.databricks_host
  token = var.token
}

data "databricks_node_type" "smallest" {
  local_disk = true
}

data "databricks_spark_version" "latest_lts" {
  long_term_support = true
}

resource "databricks_cluster" "shared_autoscaling" {
  cluster_name            = "Shared Autoscaling"
  spark_version           = data.databricks_spark_version.latest_lts.id
  node_type_id            = data.databricks_node_type.smallest.id
  autotermination_minutes = 20
  autoscale {
    min_workers = 1
    max_workers = 3
  }
  custom_tags = {
    "Owner" = "Terraform Cloud"
  }
}


output "terraform_workspace_id" {
  value = terraform.workspace
}

output "cluster_id" {
  value = databricks_cluster.shared_autoscaling.id
  
}
```
5. Go to terraform cloud and create a workspace https://app.terraform.io/app
  chose API Driven Workflow
  Name demo-databricks
6. Add .github/workflows folder and add ci cd files
    https://developer.hashicorp.com/terraform/tutorials/automation/github-actions

```yml
# terraform-plan.yml
name: "Terraform Plan"

on:
  pull_request:

env:
  TF_CLOUD_ORGANIZATION: "pedrojunqueiraio"
  TF_API_TOKEN: "${{ secrets.TF_API_TOKEN }}"
  TF_WORKSPACE: "demo-databricks"
  CONFIG_DIRECTORY: "./"

jobs:
  terraform:
    if: github.repository != 'hashicorp-education/learn-terraform-github-actions'
    name: "Terraform Plan"
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Upload Configuration
        uses: hashicorp/tfc-workflows-github/actions/upload-configuration@v1.0.0
        id: plan-upload
        with:
          workspace: ${{ env.TF_WORKSPACE }}
          directory: ${{ env.CONFIG_DIRECTORY }}
          speculative: true

      - name: Create Plan Run
        uses: hashicorp/tfc-workflows-github/actions/create-run@v1.0.0
        id: plan-run
        with:
          workspace: ${{ env.TF_WORKSPACE }}
          configuration_version: ${{ steps.plan-upload.outputs.configuration_version_id }}
          plan_only: true

      - name: Get Plan Output
        uses: hashicorp/tfc-workflows-github/actions/plan-output@v1.0.0
        id: plan-output
        with:
          plan: ${{ fromJSON(steps.plan-run.outputs.payload).data.relationships.plan.data.id }}

      - name: Update PR
        uses: actions/github-script@v6
        id: plan-comment
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            // 1. Retrieve existing bot comments for the PR
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });
            const botComment = comments.find(comment => {
              return comment.user.type === 'Bot' && comment.body.includes('Terraform Cloud Plan Output')
            });
            const output = `#### Terraform Cloud Plan Output
               \`\`\`
               Plan: ${{ steps.plan-output.outputs.add }} to add, ${{ steps.plan-output.outputs.change }} to change, ${{ steps.plan-output.outputs.destroy }} to destroy.
               \`\`\`
               [Terraform Cloud Plan](${{ steps.plan-run.outputs.run_link }})
               `;
            // 3. Delete previous comment so PR timeline makes sense
            if (botComment) {
              github.rest.issues.deleteComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
              });
            }
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
```

```yml
# terraform-apply.yml
name: "Terraform Apply"

on:
  push:
    branches:
      - master

env:
  TF_CLOUD_ORGANIZATION: "pedrojunqueiraio"
  TF_API_TOKEN: "${{ secrets.TF_API_TOKEN }}"
  TF_WORKSPACE: "demo-databricks"
  CONFIG_DIRECTORY: "./"

jobs:
  terraform:
    if: github.repository != 'hashicorp-education/learn-terraform-github-actions'
    name: "Terraform Apply"
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Upload Configuration
        uses: hashicorp/tfc-workflows-github/actions/upload-configuration@v1.0.0
        id: apply-upload
        with:
          workspace: ${{ env.TF_WORKSPACE }}
          directory: ${{ env.CONFIG_DIRECTORY }}

      - name: Create Apply Run
        uses: hashicorp/tfc-workflows-github/actions/create-run@v1.0.0
        id: apply-run
        with:
          workspace: ${{ env.TF_WORKSPACE }}
          configuration_version: ${{ steps.apply-upload.outputs.configuration_version_id }}

      - name: Apply
        uses: hashicorp/tfc-workflows-github/actions/apply-run@v1.0.0
        if: fromJSON(steps.apply-run.outputs.payload).data.attributes.actions.IsConfirmable
        id: apply
        with:
          run: ${{ steps.apply-run.outputs.run_id }}
          comment: "Apply Run from GitHub Actions CI ${{ github.sha }}"
```

7. create DBX workspace token
8. add DBX token and host in terraform cloud workspace variables
  databricks_host = "https://adb-3304201807713135.15.azuredatabricks.net"
  token           = "dapiXXXXXXX"
9. create TF_API_TOKEN in terraform cloud
  https://app.terraform.io/app/settings/tokens
10. add token in git hub secrets
11. commit and push
12. open pull request
13. merge
14. check if resource was deployed in DBX
15. plan and destroy from terraform cloud