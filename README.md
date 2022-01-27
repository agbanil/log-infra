# Log Infrastructure

what do we have here?

- [Infrastructure](#Infrastructure)
    - [Modules](#Modules)
    - [external_dns](#external_dns)
    - [null_resources](#null_resources)
    - [remote_state](#remote_state)
    - [Files](#Files)
- [Running The Infrastructure Setup (Putting it all together)](#Running-The-Infrastructure-Setup-Putting-it-all-together)
- [Proposed CI/CD setup for infrastructure](#Proposed-CI-CD-setup-for-infrastructure)
- [Improvements](#Improvements)

## Infrastructure
The infrastructure was built using Terraform for IAC. Points considered when designing this setup were, production readiness, scalability and maintainability. This setup spins up a kubernetes cluster on AWS EKS along with zones, vpc, etc., necessary to work with the cluster. Although this infrastructure is meant to house the log analyzer application (which is a daemonset), I took the liberty to setup other components necessary for it to house applications that would also need to be accessed from outside the cluster. The folder structure is as follows:
```
log-infra
├── modules
│   └── external_dns
│   │   └── main.tf
│   │   └── variables.tf
│   └── null_resources
│   │   └── main.tf
│   │   └── variables.tf
│   └── remote_state
│       └── main.tf
│       └── variables.tf
└── data.tf
└── locals.tf
└── main.tf
└── output-eks.tf
└── output-zones.tf
└── providers.tf
└── variables-eks.tf
└── variables-vpc.tf
```

### Modules
Default modules were used in this setup (zones, vpc, eks) however, some custom modules were created as well:
### external_dns
This ensures that whenever we have an application specific ingress with a subdomain specified, this record will be automatically created in AWS Route53 if it doesn't exist. It uses the helm provider as it's a helm chart applied to the EKS cluster.
### null_resources
This enables us to run regular bash commands as it uses the `local-exec` provisioner.
### remote_state
This enables us to have a synchronized terraform state across members of a team using terraform. Now, team members working with local states can be problematic when everyone has different states of the same infrastructure on their local machines. The remote state module helps mitigate this by creating what's needed to store and lock the terraform state on AWS which are, an s3 bucket and a dynamo db table. How this works is, when someone newly clones the repo, during the initialization process (`terraform init`), if there's a remote state configured, it'll be used.
### Files
- The `data` files generally hold data from data sources.
- The `variables-*` files were created as such so as to be able to make clear what variable is for what. This makes it easier when working with variables in the codebase.
- Same deal with the `output-*` files. We can easily distinguish what output is for what.
- We have three `providers` setup, `helm`, `kubernetes` and `aws`. The helm one helps with applying the `external_dns` helm chart and any other helm chart in the future into the cluster.

## Running The Infrastructure Setup (Putting it all together)
In order to run the infrastructure bit, you need these:
- Terraform version 13.3
- AWS cli configured with AWS credentials that have access to an AWS account with some credits.
- kubectl
- Go inside the `main.tf` in the root directory and change the value for the `zone` module from `k8s.lawrencetalks.com` to a subdomain you manage

Then do `terraform init` to initialize terraform. This will pull in the required modules. Next:
- Do `terraform plan` to view the changes you're about to make to the your AWS infra
- Do `terraform apply` to make the changes. This spins up the kubernetes cluster with the vpc, zones etc., sets up remote state and updates your `kubeconfig` with entries for the newly created cluster. Also from the outputs, check for the `ns` records in the `this_route53_zone_name_servers` output from the created zone and go update it over at your domain registrar in order to have AWS manage that subdomain.

Note: for a first time setup, after doing `terraform apply`, you'll get a `backend.tf` file generated. Please commit this file as this helps with working with the remote state.


## Proposed CI/CD setup for infrastructure
The infrastructure's pipeline would have the following steps:
- A `linting` step that would run `terraform validate` to check syntax etc.
- A step for `terraform plan` where the output can be sent to say, a slack channel
- A `apply` step that pushes the validated update to the infrastructure.

## Improvements
- Having a pipeline for the infrastructure
- Move the nginx ingress step to an actual terraform helm step (like the way external dns is at the moment)
