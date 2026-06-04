# Requirements Document

## Introduction

This feature delivers a set of Azure Policy definitions, initiatives (policy sets), and assignment templates that enforce the five pillars of the Azure Well-Architected Framework (WAF) across the Customer Directory Service landing zone. The landing zone follows an Azure Landing Zone (ALZ) hub-and-spoke topology with a Management Group hierarchy, Azure Firewall Premium, Application Gateway v2 + WAF, AKS (private cluster, Azure CNI), Key Vault, ACR, and Azure Monitor — all accessed via private endpoints.

Policies use built-in Azure Policy definitions wherever they exist. Custom policy definitions are created only where no built-in covers the requirement. Every policy references the WAF pillar and principle it enforces. All definitions, initiatives, and assignment templates are expressed as Bicep or ARM/JSON files under `infra/policies/`.

---

## Glossary

- **Policy_Engine**: The Azure Policy service that evaluates, audits, and enforces policy definitions against Azure resources.
- **Policy_Definition**: A single Azure Policy rule expressed as a JSON document with a `policyRule`, `parameters`, and metadata.
- **Initiative**: An Azure Policy Set Definition (policy initiative) that groups related Policy_Definitions under a single assignment unit.
- **Assignment**: An Azure Policy Assignment that binds an Initiative or Policy_Definition to a Management Group or Subscription scope.
- **Parameters_File**: A JSON file that supplies environment-specific values (e.g., prod vs non-prod) to an Assignment at deploy time.
- **WAF_Pillar**: One of the five Azure Well-Architected Framework pillars: Reliability, Security, Cost_Optimisation, Operational_Excellence, or Performance_Efficiency.
- **Management_Group**: An Azure Management Group node in the hierarchy (Tenant Root → Platform MG / Workloads MG → Subscriptions).
- **Workloads_MG**: The Management Group that contains the Production and Non-Prod subscriptions for the Customer Directory Service.
- **Built_In_Policy**: An Azure Policy definition maintained by Microsoft and referenced by its built-in policy definition ID.
- **Custom_Policy**: A Policy_Definition authored in this repository because no Built_In_Policy covers the requirement.
- **Remediation_Task**: An Azure Policy remediation that brings non-compliant resources into compliance via a `deployIfNotExists` or `modify` effect.
- **Log_Analytics_Workspace**: The central Azure Log Analytics Workspace in the Management Subscription that receives all diagnostic logs.
- **AKS_Cluster**: The Azure Kubernetes Service private cluster in the spoke VNet that hosts the customer-api workload.
- **Key_Vault**: The Azure Key Vault instance in the Identity Subscription that stores the API_KEY secret and TLS certificates.
- **ACR**: The Azure Container Registry Premium instance that stores the customer-directory container image.
- **Defender_for_Cloud**: Microsoft Defender for Cloud, including Defender for Containers, enabled across all subscriptions.
- **NSG**: A Network Security Group applied to each subnet in the spoke VNet.
- **Private_Endpoint**: An Azure Private Endpoint that exposes a PaaS service (Key Vault, ACR, Azure Monitor) on a private VNet IP.
- **Resource_Lock**: An Azure Management Lock (`CanNotDelete` or `ReadOnly`) applied to production resources to prevent accidental deletion or modification.
- **Diagnostic_Setting**: An Azure resource diagnostic setting that routes platform logs and metrics to the Log_Analytics_Workspace.
- **HPA**: Kubernetes Horizontal Pod Autoscaler that scales the customer-api Deployment based on CPU/memory metrics.

---

## Requirements

### Requirement 1: Reliability — Availability Zones

**User Story:** As a platform engineer, I want Azure Policy to enforce availability zone deployment for supported resources, so that the Customer Directory Service landing zone is resilient to single-zone failures.

#### Acceptance Criteria

1. WHEN a supported resource type (AKS node pool, Application Gateway v2, Azure Firewall) is created or updated without availability zone configuration, THE Policy_Engine SHALL deny the operation with a descriptive error message referencing the WAF Reliability pillar.
2. THE Policy_Engine SHALL audit existing resources that are not deployed across at least two availability zones and report them as non-compliant.
3. WHERE the resource type does not support availability zones in the target region, THE Policy_Engine SHALL exempt the resource from the availability zone requirement.

---

### Requirement 2: Reliability — Resource Locks on Production

**User Story:** As a platform engineer, I want Azure Policy to enforce resource locks on production resources, so that critical infrastructure cannot be accidentally deleted or modified.

#### Acceptance Criteria

1. WHEN a resource in the Production Subscription is created without a `CanNotDelete` Management Lock, THE Policy_Engine SHALL deploy a `CanNotDelete` lock via a `deployIfNotExists` effect.
2. THE Policy_Engine SHALL scope the lock enforcement to resource groups tagged with `environment: prod`.
3. IF a `CanNotDelete` lock is removed from a production resource group, THEN THE Policy_Engine SHALL re-apply the lock via a Remediation_Task.

---

### Requirement 3: Reliability — Health Probes on Load Balancers and Application Gateway

**User Story:** As a platform engineer, I want Azure Policy to audit that health probes are configured on Application Gateway backend pools, so that unhealthy backends are automatically removed from rotation.

#### Acceptance Criteria

1. THE Policy_Engine SHALL audit Application Gateway v2 instances that have backend pools without a custom health probe configured and report them as non-compliant.
2. WHEN an Application Gateway v2 is created or updated with a backend pool that has no health probe, THE Policy_Engine SHALL emit an audit event to the Log_Analytics_Workspace.

---

### Requirement 4: Reliability — Autoscale on AKS Node Pools

**User Story:** As a platform engineer, I want Azure Policy to enforce that AKS user node pools have cluster autoscaler enabled, so that the workload can scale to meet demand without manual intervention.

#### Acceptance Criteria

1. WHEN an AKS_Cluster user node pool is created or updated with `enableAutoScaling: false`, THE Policy_Engine SHALL deny the operation.
2. THE Policy_Engine SHALL audit existing AKS_Cluster user node pools where autoscaling is disabled and report them as non-compliant.
3. THE Policy_Engine SHALL not apply the autoscale requirement to AKS system node pools.

---

### Requirement 5: Security — Private Endpoints for PaaS Services

**User Story:** As a security officer, I want Azure Policy to enforce that Key Vault, ACR, and Azure Monitor use private endpoints, so that PaaS services are never reachable over the public internet.

#### Acceptance Criteria

1. WHEN a Key_Vault instance is created or updated without a Private_Endpoint connection in an approved state, THE Policy_Engine SHALL audit the resource as non-compliant.
2. WHEN an ACR instance is created or updated without a Private_Endpoint connection in an approved state, THE Policy_Engine SHALL audit the resource as non-compliant.
3. THE Policy_Engine SHALL use the Built_In_Policy `[Preview]: Azure Key Vault should use private link` (ID: `a6abeaec-4d90-4a02-805f-6b26c4d3fbe9`) for Key Vault private endpoint enforcement.
4. THE Policy_Engine SHALL use the Built_In_Policy `Container registries should use private link` (ID: `e8eef0a8-67cf-4eb4-9386-14b0e78733d4`) for ACR private endpoint enforcement.

---

### Requirement 6: Security — Deny Public IP Addresses

**User Story:** As a security officer, I want Azure Policy to deny the creation of public IP addresses on resources in the spoke VNet subscriptions, so that workload resources are never directly exposed to the internet.

#### Acceptance Criteria

1. WHEN a public IP address resource is created in the Workloads_MG scope with a SKU other than `Basic` and not tagged with `public-ip-exemption: true`, THE Policy_Engine SHALL deny the operation.
2. THE Policy_Engine SHALL allow public IP addresses that are explicitly associated with the Application Gateway v2 or Azure Firewall in the Connectivity Subscription (hub VNet), as those are in the Platform MG scope.
3. IF a public IP address is created without the exemption tag in the Workloads_MG scope, THEN THE Policy_Engine SHALL log the denial event to the Log_Analytics_Workspace.

---

### Requirement 7: Security — Require HTTPS and TLS 1.2+

**User Story:** As a security officer, I want Azure Policy to enforce HTTPS-only access and TLS 1.2 or higher on all applicable services, so that data in transit is always encrypted.

#### Acceptance Criteria

1. WHEN an Application Gateway v2 listener is configured with HTTP (not HTTPS), THE Policy_Engine SHALL audit the resource as non-compliant.
2. THE Policy_Engine SHALL use the Built_In_Policy `App Service apps should only be accessible over HTTPS` where applicable, and a Custom_Policy for Application Gateway HTTPS listener enforcement.
3. WHEN a Key_Vault instance is created or updated with a minimum TLS version below `TLS1_2`, THE Policy_Engine SHALL deny the operation.
4. THE Policy_Engine SHALL use the Built_In_Policy `Key vaults should have the minimum TLS version of 1.2` (ID: `1f314764-cb73-4fc9-b863-8eca98ac36e9`) for Key Vault TLS enforcement.

---

### Requirement 8: Security — Key Vault Soft-Delete and Purge Protection

**User Story:** As a security officer, I want Azure Policy to enforce that Key Vault has soft-delete and purge protection enabled, so that secrets cannot be permanently deleted without a recovery window.

#### Acceptance Criteria

1. WHEN a Key_Vault instance is created or updated with `softDeleteRetentionInDays` less than `90`, THE Policy_Engine SHALL deny the operation.
2. WHEN a Key_Vault instance is created or updated with `enablePurgeProtection: false`, THE Policy_Engine SHALL deny the operation.
3. THE Policy_Engine SHALL use the Built_In_Policy `Key vaults should have soft delete enabled` (ID: `1e66c121-a66a-4b1f-9b83-0fd99bf0fc2d`) for soft-delete enforcement.
4. THE Policy_Engine SHALL use the Built_In_Policy `Key vaults should have purge protection enabled` (ID: `0b60c0b2-2dc2-4e1c-b5c9-abbed971de53`) for purge protection enforcement.

---

### Requirement 9: Security — Defender for Cloud Plans Enabled

**User Story:** As a security officer, I want Azure Policy to enforce that Microsoft Defender for Cloud plans are enabled on all subscriptions, so that threat detection and CSPM coverage is never inadvertently disabled.

#### Acceptance Criteria

1. WHEN a subscription in the Workloads_MG does not have Defender for Containers enabled, THE Policy_Engine SHALL audit the subscription as non-compliant.
2. THE Policy_Engine SHALL use the Built_In_Policy `Microsoft Defender for Containers should be enabled` (ID: `1c988dd6-ade4-430f-a608-2a3e5b0a6d38`) for Defender for Containers enforcement.
3. THE Policy_Engine SHALL audit subscriptions where the Defender for Cloud security score is below the configured threshold and report them as non-compliant.

---

### Requirement 10: Security — No Privileged Containers

**User Story:** As a security officer, I want Azure Policy to deny privileged containers in AKS, so that workload pods cannot escalate privileges to the node level.

#### Acceptance Criteria

1. WHEN a Kubernetes Pod or Deployment is created in the AKS_Cluster with `securityContext.privileged: true`, THE Policy_Engine SHALL deny the operation.
2. THE Policy_Engine SHALL use the Built_In_Policy `Kubernetes cluster should not allow privileged containers` (ID: `95edb821-ddaf-4404-9732-666045e70207`) for privileged container enforcement.
3. THE Policy_Engine SHALL apply the privileged container denial to all namespaces except `kube-system` and `gatekeeper-system`.

---

### Requirement 11: Security — Approved Container Registries Only

**User Story:** As a security officer, I want Azure Policy to restrict AKS pods to pulling images only from the approved ACR instance, so that untrusted or public images cannot run in the cluster.

#### Acceptance Criteria

1. WHEN a Kubernetes Pod is created in the AKS_Cluster with an image that does not originate from the approved ACR hostname (e.g., `customerdirectoryprod.azurecr.io`), THE Policy_Engine SHALL deny the operation.
2. THE Policy_Engine SHALL use the Built_In_Policy `Kubernetes cluster containers should only use allowed images` (ID: `febd0533-8e55-448f-b837-bd0e06f16469`) for approved registry enforcement.
3. THE Parameters_File SHALL supply the approved ACR hostname as a parameter so that prod and non-prod environments can reference different registries without modifying the policy definition.

---

### Requirement 12: Cost Optimisation — Mandatory Resource Tagging

**User Story:** As a finance officer, I want Azure Policy to enforce that all resources carry mandatory cost allocation tags, so that cloud spend can be attributed to cost centres, environments, and owners.

#### Acceptance Criteria

1. WHEN a resource is created or updated without the `costCentre` tag, THE Policy_Engine SHALL deny the operation with a message identifying the missing tag.
2. WHEN a resource is created or updated without the `environment` tag, THE Policy_Engine SHALL deny the operation with a message identifying the missing tag.
3. WHEN a resource is created or updated without the `owner` tag, THE Policy_Engine SHALL deny the operation with a message identifying the missing tag.
4. THE Policy_Engine SHALL use the Built_In_Policy `Require a tag on resources` (ID: `871b6d14-10aa-478d-b590-94f262ecfa99`) applied three times — once per required tag — for tag enforcement.
5. THE Parameters_File SHALL define the allowed values for the `environment` tag (e.g., `["prod", "dev", "test"]`) so that free-text environment values are rejected.

---

### Requirement 13: Cost Optimisation — Budget Alert Enforcement

**User Story:** As a finance officer, I want Azure Policy to audit that Azure Budget alerts are configured on all subscriptions, so that unexpected cost spikes trigger notifications before they become significant.

#### Acceptance Criteria

1. THE Policy_Engine SHALL audit subscriptions in the Workloads_MG that do not have at least one Azure Budget with an alert threshold configured and report them as non-compliant.
2. WHEN a subscription has no Budget resource, THE Policy_Engine SHALL emit an audit event to the Log_Analytics_Workspace.
3. THE Policy_Engine SHALL use a Custom_Policy for budget alert enforcement, as no Built_In_Policy covers subscription-level budget existence checks.

---

### Requirement 14: Cost Optimisation — Right-Sizing Guardrails

**User Story:** As a finance officer, I want Azure Policy to restrict VM and AKS node pool SKUs to an approved list, so that over-provisioned or expensive SKUs cannot be deployed without explicit approval.

#### Acceptance Criteria

1. WHEN an AKS node pool is created or updated with a VM SKU not in the approved SKU list, THE Policy_Engine SHALL deny the operation.
2. THE Policy_Engine SHALL use the Built_In_Policy `Allowed virtual machine size SKUs` (ID: `cccc23c7-8427-4f53-ad12-b6a63eb452b3`) for VM SKU restriction.
3. THE Parameters_File SHALL define the approved SKU list (e.g., `["Standard_D2s_v5", "Standard_D4s_v5", "Standard_D8s_v5"]`) so that the list can differ between prod and non-prod without modifying the policy definition.

---

### Requirement 15: Operational Excellence — Diagnostic Settings to Log Analytics

**User Story:** As an operator, I want Azure Policy to enforce that all supported resources emit diagnostic logs to the central Log Analytics Workspace, so that no audit trail is lost and all logs are queryable from a single location.

#### Acceptance Criteria

1. WHEN a supported resource (AKS_Cluster, Key_Vault, ACR, Application Gateway, Azure Firewall) is created or updated without a Diagnostic_Setting pointing to the Log_Analytics_Workspace, THE Policy_Engine SHALL deploy the Diagnostic_Setting via a `deployIfNotExists` effect.
2. THE Policy_Engine SHALL use the Built_In_Policy `Deploy Diagnostic Settings for Key Vault to Log Analytics workspace` (ID: `bef3f64c-5290-43b7-85b0-9b254eef4c47`) for Key Vault diagnostic settings.
3. THE Policy_Engine SHALL use the Built_In_Policy `Deploy Diagnostic Settings for Azure Kubernetes Service to Log Analytics workspace` (ID: `6c66c325-74c8-42fd-a286-a74b0e2939d8`) for AKS diagnostic settings.
4. THE Parameters_File SHALL supply the Log_Analytics_Workspace resource ID as a parameter so that prod and non-prod environments can target different workspaces.
5. WHEN a Remediation_Task is triggered for a non-compliant resource, THE Policy_Engine SHALL deploy the Diagnostic_Setting within 30 minutes of the remediation task execution.

---

### Requirement 16: Operational Excellence — Naming Convention Enforcement

**User Story:** As a platform engineer, I want Azure Policy to audit that resources follow the approved naming convention, so that resource names are consistent and discoverable across the landing zone.

#### Acceptance Criteria

1. THE Policy_Engine SHALL audit resources whose names do not match the approved prefix pattern for their resource type (e.g., AKS clusters must begin with `aks-`, Key Vaults with `kv-`, resource groups with `rg-`) and report them as non-compliant.
2. THE Policy_Engine SHALL use a Custom_Policy with a `match` condition on the resource name for naming convention enforcement, as no Built_In_Policy covers custom prefix patterns.
3. THE Policy_Engine SHALL apply the naming convention audit at the Workloads_MG scope so that both prod and non-prod subscriptions are covered.

---

### Requirement 17: Operational Excellence — Activity Log Retention

**User Story:** As a compliance officer, I want Azure Policy to enforce that the Azure Activity Log is retained for at least 365 days, so that audit trails meet public-sector retention requirements.

#### Acceptance Criteria

1. WHEN a subscription's Activity Log diagnostic setting has a retention period less than `365` days, THE Policy_Engine SHALL audit the subscription as non-compliant.
2. THE Policy_Engine SHALL use the Built_In_Policy `Activity log should be retained for at least one year` (ID: `b02aacc0-b073-424e-8298-42b22829ee0a`) for activity log retention enforcement.
3. THE Policy_Engine SHALL apply the activity log retention policy at the Workloads_MG scope.

---

### Requirement 18: Performance Efficiency — AKS Autoscale Enforcement

**User Story:** As a platform engineer, I want Azure Policy to enforce that AKS user node pools have autoscaling enabled with defined minimum and maximum node counts, so that the cluster can respond to load changes without manual intervention.

#### Acceptance Criteria

1. WHEN an AKS_Cluster user node pool is created or updated with `enableAutoScaling: false`, THE Policy_Engine SHALL deny the operation (this requirement aligns with Requirement 4 and the same policy definition covers both pillars).
2. WHEN an AKS_Cluster user node pool is created or updated with `minCount` less than `2`, THE Policy_Engine SHALL deny the operation to ensure baseline availability.
3. THE Policy_Engine SHALL use a Custom_Policy for the `minCount` enforcement, as no Built_In_Policy covers AKS node pool minimum count.

---

### Requirement 19: Performance Efficiency — Container Resource Requests and Limits

**User Story:** As a platform engineer, I want Azure Policy to enforce that all containers in AKS define CPU and memory resource requests and limits, so that the Kubernetes scheduler can make accurate placement decisions and prevent noisy-neighbour resource exhaustion.

#### Acceptance Criteria

1. WHEN a Kubernetes Pod is created in the AKS_Cluster with a container that has no CPU resource request defined, THE Policy_Engine SHALL deny the operation.
2. WHEN a Kubernetes Pod is created in the AKS_Cluster with a container that has no memory resource request defined, THE Policy_Engine SHALL deny the operation.
3. WHEN a Kubernetes Pod is created in the AKS_Cluster with a container that has no CPU resource limit defined, THE Policy_Engine SHALL deny the operation.
4. WHEN a Kubernetes Pod is created in the AKS_Cluster with a container that has no memory resource limit defined, THE Policy_Engine SHALL deny the operation.
5. THE Policy_Engine SHALL use the Built_In_Policy `Kubernetes cluster containers should not use forbidden sysctl interfaces` and `Kubernetes clusters should not allow container privilege escalation` as complementary controls, and a Custom_Policy for resource request/limit enforcement.
6. THE Policy_Engine SHALL exempt the `kube-system` and `gatekeeper-system` namespaces from resource request/limit enforcement.

---

### Requirement 20: Policy Initiative Structure

**User Story:** As a platform engineer, I want all policies grouped into WAF-pillar initiatives, so that I can assign, report on, and manage compliance by pillar rather than by individual policy.

#### Acceptance Criteria

1. THE Policy_Engine SHALL group all Reliability policies (Requirements 1–4) into a single Initiative named `WAF-Reliability-Initiative`.
2. THE Policy_Engine SHALL group all Security policies (Requirements 5–11) into a single Initiative named `WAF-Security-Initiative`.
3. THE Policy_Engine SHALL group all Cost Optimisation policies (Requirements 12–14) into a single Initiative named `WAF-CostOptimisation-Initiative`.
4. THE Policy_Engine SHALL group all Operational Excellence policies (Requirements 15–17) into a single Initiative named `WAF-OperationalExcellence-Initiative`.
5. THE Policy_Engine SHALL group all Performance Efficiency policies (Requirements 18–19) into a single Initiative named `WAF-PerformanceEfficiency-Initiative`.
6. WHEN an Initiative is assigned to a Management Group or Subscription, THE Policy_Engine SHALL surface per-policy compliance results within the initiative compliance view.

---

### Requirement 21: Assignment Templates

**User Story:** As a platform engineer, I want Bicep or ARM assignment templates for Management Group and Subscription scope, so that initiatives can be deployed consistently via CI/CD without manual portal configuration.

#### Acceptance Criteria

1. THE Assignment template SHALL support assignment at both Management Group scope and Subscription scope via a parameter, without requiring separate template files.
2. THE Assignment template SHALL reference the Parameters_File to supply environment-specific values (Log Analytics Workspace ID, approved ACR hostname, approved VM SKUs, allowed environment tag values).
3. WHEN the Assignment template is deployed with `effect: Disabled`, THE Policy_Engine SHALL evaluate but not enforce the policy, enabling dry-run validation before enforcement.
4. THE Assignment template SHALL include a system-assigned managed identity with the minimum required RBAC roles for `deployIfNotExists` and `modify` policy effects.
5. THE Assignment template SHALL set `enforcementMode: Default` for production assignments and `enforcementMode: DoNotEnforce` for non-prod assignments, controlled via the Parameters_File.

---

### Requirement 22: Folder Structure and Deliverable Organisation

**User Story:** As a platform engineer, I want all policy files organised under `infra/policies/` with a clear folder structure, so that the repository is navigable and CI/CD pipelines can target specific pillars independently.

#### Acceptance Criteria

1. THE Policy_Engine artefacts SHALL be organised under `infra/policies/` with sub-folders per WAF pillar: `reliability/`, `security/`, `cost-optimisation/`, `operational-excellence/`, `performance-efficiency/`.
2. THE `infra/policies/` directory SHALL contain an `initiatives/` sub-folder with one initiative definition file per WAF pillar.
3. THE `infra/policies/` directory SHALL contain an `assignments/` sub-folder with the assignment template and the Parameters_File.
4. THE `infra/policies/` directory SHALL contain a `README.md` that documents the folder structure, lists all policies by pillar, identifies which are Built_In_Policy references vs Custom_Policy definitions, and provides deployment instructions.
5. WHEN a new Custom_Policy is added, THE Policy_Definition file SHALL include a `metadata` block with `wafPillar`, `wafPrinciple`, `policyType: Custom`, and `version` fields.
