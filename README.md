# HumanGov — Infrastructure (Terraform, Ansible *Legacy*)

Terraform for HumanGov, a multi-tenant SaaS HR application deployed on Amazon
EKS. A reusable module provisions an isolated DynamoDB table and S3 bucket for
each U.S. state, all driven by a single list of states.

→ [EKS Deployment](https://medium.com/@ivantrevino/humangov-deployment-of-humangov-saas-application-on-aws-elastic-kubernetes-service-eks-using-a-da0f64ab9ad9) · [CI/CD Pipeline](https://medium.com/@ivantrevino/humangov-automating-humangov-saas-application-build-and-deployment-process-on-kubernetes-with-a9f167546fad)

Application code, Kubernetes manifests, and pipeline details live in the
companion repo: human-gov-app.

---

## What This Does

Each state is an isolated tenant with its own data stores. The root
configuration calls one module once per entry in `states`. Adding a state is a
one-line change plus `terraform apply`, and existing states are left
untouched.

The outputs return a map of each state to its table and bucket names. Those
values go into the state's Kubernetes manifest in human-gov-app as
`AWS_DYNAMODB_TABLE` and `AWS_BUCKET`.

Current states: california, florida, and staging. Staging backs the
pipeline's staging environment ahead of manual approval.

---

## Architecture

Provisioned by this repo, per state:

- **DynamoDB table** — `humangov-<state>-dynamodb`, on-demand billing, hash key `id` (string). Stores employee records.
- **S3 bucket** — `humangov-<state>-s3-<suffix>`. Stores employee ID documents (PDF). A 4-character `random_string` suffix keeps the name globally unique.
- **Tags** — every resource is tagged with the state name

Provisioned outside this repo (see human-gov-app):

- EKS cluster, AWS Load Balancer Controller, IRSA service accounts (eksctl and Helm)
- ECR repository, CodePipeline, and CodeBuild projects
- Route53 records and the ACM certificate
- CloudWatch Synthetics canaries and the DynamoDB Stream → Lambda microservice

---

## Repository Structure

    terraform/
      main.tf          Calls the module for each state in var.states
      variables.tf     states list, region
      outputs.tf       state => { dynamodb_table, s3_bucket }
    modules/
      aws_humangov_infrastructure/
        main.tf        DynamoDB table, random suffix, S3 bucket
        variables.tf   state_name, region
        outputs.tf     state_dynamodb_table, state_s3_bucket
    ansible/           Legacy: humangov-webapp role from the EC2 phase,
                       not used by the EKS deployment

---

## How to Run

    cd terraform
    terraform init
    terraform plan
    terraform apply
    terraform output

To add a state, append it to `states` in `variables.tf`:

    variable "states" {
      description = "The list of state names"
      default     = ["california", "florida", "staging"]
    }

Then apply, and copy the new outputs into that state's manifest in
human-gov-app.

---

## Evolution

The module started as a full per-state stack for the EC2 phase of the project:
an EC2 instance, security group, IAM role and instance profile, and
`local-exec` provisioners that kept the Ansible inventory in sync. When
HumanGov moved to containers (ECS, then EKS), compute moved off EC2 instances
and app permissions moved to an IRSA service account. Those blocks are now
commented out, and the module provisions only the data layer.

---

## Known Limitations

- The commented-out EC2 blocks contain a hardcoded security group ID and AMI ID. Remove them or turn them into variables before reuse.
- Buckets have no explicit encryption, versioning, or public access block resources. Production should define these in the module.

---

Originally hosted on AWS CodeCommit, migrated to GitHub.
