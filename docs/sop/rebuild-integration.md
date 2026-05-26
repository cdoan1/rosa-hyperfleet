# Rebuild the Integration Environment

Tear down and re-provision the integration environment when it is unrecoverable through normal pipeline runs.

## Prerequisites

1. AWS Console access to the **central account** (hosts CodePipeline)
2. AWS Console or CLI access to the **RC and MC target accounts** (for manual cleanup of leftover resources)
3. [Vault](https://vault.ci.openshift.org) access to update CI credentials after rebuild

## Procedure

The rebuild follows a teardown-then-provision cycle. MC must be torn down before RC because it depends on RC resources (IoT certificates, DNS zones).

```mermaid
sequenceDiagram
    participant Operator
    participant Console as AWS Console<br/>(Central Account)
    participant MC as MC Pipeline<br/>(mc01-pipe)
    participant RC as RC Pipeline<br/>(regional-pipe)
    participant Vault

    Operator->>Console: Start MC pipeline (IS_DESTROY=true)
    Console->>MC: Trigger teardown
    MC-->>Console: Complete (may partially fail)
    Operator->>Console: Start RC pipeline (IS_DESTROY=true)
    Console->>RC: Trigger teardown
    RC-->>Console: Complete (may partially fail)
    Operator->>Console: Manually delete leftover resources
    Note over Operator,Console: Check RC and MC accounts for<br/>orphaned resources
    Operator->>Console: Re-run provisioner pipeline
    Console->>RC: Provision RC
    RC-->>Console: Complete
    Console->>MC: Provision MC
    MC-->>Console: Complete
    Operator->>Vault: Update api_url with new API Gateway URL
    Operator->>Operator: Trigger nightly-integration to verify
```

### Step 1: Tear down the Management Cluster

1. Open the [AWS CodePipeline console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines) in the **central account**.

2. Find the MC pipeline: `mc01-pipe`.

3. Click **Release change** (or **Start execution** in the pipeline detail view). When prompted for pipeline variables, set:

   ```
   IS_DESTROY = true
   ```

4. Wait for the pipeline to complete. The teardown may partially fail due to dependency violations (e.g., RDS ENIs, EKS-managed resources). This is expected; leftover resources are cleaned up in Step 3.

### Step 2: Tear down the Regional Cluster

1. In the same CodePipeline console, find the RC pipeline: `regional-pipe`.

2. Start execution with:

   ```
   IS_DESTROY = true
   ```

3. Wait for the pipeline to complete. As with the MC, partial failures are expected.

### Step 3: Clean up leftover resources

After both teardowns complete, some resources may remain in the RC and MC AWS accounts. Switch to each target account and check for:

- **Secrets Manager secrets** with deletion protection or retention periods
- **RDS snapshots** (final snapshots created during instance deletion)
- **ENIs** attached to RDS or other managed services
- **EBS volumes** that were not deleted with their instances
- **VPC resources** (subnets, internet gateways, NAT gateways, security groups, the VPC itself) if the VPC teardown failed due to dependencies
- **EKS clusters** or node groups that were not fully deleted

Delete these manually via the AWS Console or CLI. The re-provisioning step will fail if conflicting resources remain (e.g., a VPC with the same CIDR or a security group with the same name).

Tip: to find remaining resources quickly, use [AWS Resource Explorer](https://resource-explorer.console.aws.amazon.com/) in each account.

### Step 4: Re-provision

Trigger the parent provisioner pipeline to re-create the RC and MC pipelines and start a fresh provision:

1. In the CodePipeline console (central account), find the provisioner pipeline: `pipeline-provisioner`.

2. Click **Release change**. Leave all variables at their defaults (do **not** set `FORCE_DELETE_ALL_PIPELINES`).

3. The provisioner will:
   - Rebuild the RC and MC child pipelines (if needed)
   - Trigger the RC pipeline, which provisions the regional cluster infrastructure
   - Trigger the MC pipeline, which provisions the management cluster infrastructure

4. Monitor both child pipelines until they succeed. The full provision typically takes 30-45 minutes.

   If the provision fails due to leftover resources from Step 3, delete the conflicting resources and re-trigger the failed pipeline.

## Post-Rebuild

### Update Vault credentials

The API Gateway URL changes on every rebuild. The nightly-integration CI job reads it from the Vault secret. Update it so CI tests pass:

1. Get the new API Gateway URL from the RC pipeline's Terraform output. Connect to the RC bastion and run:

   ```bash
   # From the RC bastion (make int-bastion-rc)
   # Or read from the Terraform state in the RC account:
   aws s3 cp s3://terraform-state-<RC_ACCOUNT_ID>-us-east-1/regional-cluster/regional.tfstate - \
     | jq -r '.outputs.api_gateway_invoke_url.value'
   ```

2. Update the Vault secret at [`selfservice/cluster-secrets-rosa-regional-platform-int/integration-creds`](https://vault.ci.openshift.org/ui/vault/secrets/kv/kv/list/selfservice/cluster-secrets-rosa-regional-platform-int/):
   - Edit the `api_url` field with the new API Gateway URL

### Verify the rebuild

1. Connect to the RC and MC bastions and verify ArgoCD applications are synced:

   ```bash
   make int-bastion-rc
   # On bastion:
   kubectl get applications -A
   ```

   All applications should show `Synced` and `Healthy`.

2. Test the Platform API:

   ```bash
   make int-shell
   # In shell:
   awscurl --service execute-api $API_URL/v0/live
   ```

   Expected response: `{"status":"ok"}`

3. Trigger the nightly-integration CI job to confirm end-to-end tests pass. Follow the [manual trigger instructions](../../ci/README.md#triggering-the-e2e-job-manually):

   ```bash
   curl -X POST \
       -H "Authorization: Bearer $(oc whoami -t)" \
       'https://gangway-ci.apps.ci.l2s4.p1.openshiftapps.com/v1/executions/' \
       -d '{"job_name": "periodic-ci-openshift-online-rosa-regional-platform-main-nightly-integration", "job_execution_type": "1"}'
   ```

## Reference

- [Provision a New Environment](../environment-provisioning.md) for the full provisioning guide
- [CI Jobs](../../ci/README.md) for CI job details and manual trigger instructions
- Pipeline buildspec scripts: [`scripts/buildspec/provision-infra-rc.sh`](../../scripts/buildspec/provision-infra-rc.sh), [`scripts/buildspec/provision-infra-mc.sh`](../../scripts/buildspec/provision-infra-mc.sh)
- Pipeline names are derived from `regional_id` and `management_id` in the environment config (`config/<environment>/<region>.yaml`), constructed as `<id>-pipe` in the pipeline Terraform modules
