# Supported Identity methods

Identities are used on the management cluster and the VMs/clusters/workloads which get provisioned by the management cluster.
Also see relevant [identities use cases](identities-use-cases.md), [Azure Active Directory integration](aad-integration.md), and [Multi-tenancy](multitenancy.md) pages.

## Deprecated Identity Types

<aside class="note warning">
<h1> Warning </h1>
The ability to set credentials using environment variables has been removed. Instead, use <code class="hjls">AzureClusterIdentity</code> as described below.
</aside>

<aside class="note warning">
<h1> Warning </h1>
The identity type <code class="hjls">ManualServicePrincipal</code> has been deprecated because it is now identical to <code class="hjls">ServicePrincipal</code> and therefore redundant. None of the identity types use AAD Pod Identity any longer.
</aside>

For details on the deprecated identity types, [see this page](multitenancy.md#deprecated-identity-types).

## Workload Identity (Recommended)

Follow this [link](./workload-identity.md) for a quick start guide on setting up workload identity.

Once you've set up the management cluster with the workload identity (see link above), the corresponding values should be used to create an `AzureClusterIdentity` resource. Create an `azure-cluster-identity.yaml` file with the following content:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: cluster-identity
spec:
  type: WorkloadIdentity
  tenantID: <your-tenant-id>
  clientID: <your-client-id>
  allowedNamespaces:
    list:
    - <cluster-namespace>
```

## Service Principal

Service Principal identity uses the service principal's `clientSecret` in a Kubernetes Secret. To use this type of identity, set the identity type as `ServicePrincipal` in `AzureClusterIdentity`. For example,

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: example-identity
  namespace: default
spec:
  type: ServicePrincipal
  tenantID: <azure-tenant-id>
  clientID: <client-id-of-SP-identity>
  clientSecret: {"name":"<secret-name-for-client-password>","namespace":"default"}
  allowedNamespaces:
    list:
    - <cluster-namespace>
```

Deploy this resource to your cluster:
```bash
kubectl apply -f azure-cluster-identity.yaml
```

A Kubernetes Secret should also be created to store the client password:

```bash
kubectl create secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}" --from-literal=clientSecret="${AZURE_CLIENT_SECRET}"
```

The resulting Secret should look similar to the following example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name-for-client-password>
type: Opaque
data:
  clientSecret: <client-secret-of-SP-identity>
```

## Service Principal With Certificate

Once a new SP Identity is created in Azure, the corresponding values should be used to create an `AzureClusterIdentity` resource:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: example-identity
  namespace: default
spec:
  type: ServicePrincipalCertificate
  tenantID: <azure-tenant-id>
  clientID: <client-id-of-SP-identity>
  clientSecret: {"name":"<secret-name-for-client-password>","namespace":"default"}
  allowedNamespaces:
    list:
    - <cluster-namespace>
```

If needed, convert the PEM file to PKCS12 and set a password:

```bash
openssl pkcs12 -export -in fileWithCertAndPrivateKey.pem -out ad-sp-cert.pfx -passout pass:<password>
```

Create a k8s secret with the certificate and password:

```bash
kubectl create secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}" --from-file=certificate=ad-sp-cert.pfx --from-literal=password=<password>
```

The resulting Secret should look similar to the following example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name-for-client-password>
type: Opaque
data:
  certificate: CERTIFICATE
  password: PASSWORD
```

Alternatively, the path to a certificate can be specified instead of the k8s secret:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: example-identity
  namespace: default
spec:
  type: ServicePrincipalCertificate
  tenantID: <azure-tenant-id>
  clientID: <client-id-of-SP-identity>
  certPath: <path-to-the-cert>
  allowedNamespaces:
    list:
    - <cluster-namespace>
```

## Moving identities and credential Secrets with `clusterctl move`

When you move a cluster between management clusters with `clusterctl move`, the
`AzureClusterIdentity` and the credential Secret it references are not guaranteed
to follow the cluster. Handle each of them explicitly:

### The `AzureClusterIdentity` object

`clusterctl move` carries an object that is part of a move hierarchy or that is
labeled for move. The `AzureClusterIdentity` objects in the CAPZ templates carry
the `clusterctl.cluster.x-k8s.io/move-hierarchy: "true"` label, so they move with
the cluster. An identity you created by hand does not carry the label and is not
part of the move set: after the move the moved cluster still references it, but
it does not exist on the target and CAPZ cannot reconcile the cluster. If you
want a hand-created identity to move, label it the same way:

```bash
kubectl label azureclusteridentity <identity-name> \
  clusterctl.cluster.x-k8s.io/move-hierarchy="true" --overwrite
```

### The `clientSecret` Secret

For `ServicePrincipal`, `ServicePrincipalCertificate`, and `ManualServicePrincipal`
identities, `AzureClusterIdentity.spec.clientSecret` points at a Kubernetes
Secret that stores the service principal's password (or certificate and
password). `clusterctl move` creates the Secret on the target and deletes it from
the source only when the Secret is part of the move set. For a user-created
Secret that means one of:

- the Secret is owned by an object that moves with the cluster, or
- the Secret name follows the `<cluster-name>-<purpose>` convention and the
  Secret is in the same namespace as the cluster, or
- the Secret carries the `clusterctl.cluster.x-k8s.io/move: "true"` label.

Choose the approach by how the Secret is shared:

| How the Secret is used | What to do before `clusterctl move` |
|---|---|
| Dedicated to this cluster's identity, in the cluster's namespace | Label it `clusterctl.cluster.x-k8s.io/move: "true"` (or name it `<cluster-name>-<purpose>`) so it moves with the cluster. The move deletes it from the source, which is safe because nothing else there uses it. |
| Shared by several identities or clusters, or by other consumers on the source management cluster | Do **not** label it: a moved Secret is deleted from the source, and the remaining consumers lose their credential. Create the Secret on the target instead. |
| `clientSecret.namespace` points at a namespace other than the cluster's namespace (and other than the provider namespace, `capz-system`) | `clusterctl move` does not discover Secrets in other namespaces, so a label does not help. Create the Secret on the target instead. |

Example, a dedicated Secret in the cluster's namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-cluster-identity-client-secret
  namespace: default
  labels:
    clusterctl.cluster.x-k8s.io/move: "true"
type: Opaque
stringData:
  clientSecret: <service-principal-client-secret>
```

Provisioning on the target (shared or cross-namespace cases):

```bash
# on the target management cluster, before or right after clusterctl move
kubectl -n <clientSecret-namespace> create secret generic <clientSecret-name> \
  --from-literal=clientSecret=<service-principal-client-secret>
```

`clusterctl move` deletes the moved objects from the source after creating them
on the target, so do not label a Secret that anything on the source still needs.

### Verify after the move

On the target, the moved cluster reconciles only when both pieces are present:

```bash
kubectl -n <cluster-namespace> get azureclusteridentity <identity-name>
kubectl -n <clientSecret-namespace> get secret <clientSecret-name>
kubectl -n <cluster-namespace> get cluster <cluster-name>
```

A missing credential Secret blocks reconciliation: the CAPZ controller cannot
build the per-cluster ASO credential Secret (error `failed to fetch
AzureClusterIdentity secret`), and the cluster's ASO resources stay
`Ready=False` (reason `Failed`) until the Secret exists. Wait for the `Cluster`
to report `Ready=True` (and, for AKS, for the `AzureASOManaged*` resources to be
`Ready=True`) before deleting anything on the source.

On the source, when the Secret was shared, the remaining consumers keep working
because the shared Secret was not labeled and therefore stays on the source.
Confirm the other clusters that use the same identity still reconcile:

```bash
kubectl -n <other-namespace> get cluster <other-cluster-name>
```

For the ASO credential Secret named by the `serviceoperator.azure.com/credential-from`
annotation on inline ASO resources, see
[Azure Service Operator - Migrating a cluster with `clusterctl move`](aso.md#migrating-a-cluster-with-clusterctl-move).

## User-Assigned Managed Identity

<aside class="note">

<h1> Note </h1>

This option is only available when the cluster is managed from a Kubernetes cluster running on Azure.

</aside>

#### Prerequisites

1. [Create](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp#create-a-user-assigned-managed-identity) a user-assigned managed identity in Azure.
2. [Create a role assignment](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-to-assign-access-azure-resource?pivots=identity-mi-access-portal#use-azure-rbac-to-assign-a-managed-identity-access-to-another-resource-using-the-azure-portal) to give the identity Contributor access to the Azure subscription where the workload cluster will be created.
3. Configure the identity on the management cluster nodes by adding it to each worker node VM. If using AKS as the management cluster see [these instructions](https://learn.microsoft.com/azure/aks/use-managed-identity).

#### Creating the AzureClusterIdentity

After a user-assigned managed identity is created in Azure and assigned to the management cluster, the corresponding values should be used to create an `AzureClusterIdentity` resource:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: example-identity
  namespace: default
spec:
  type: UserAssignedMSI
  tenantID: <azure-tenant-id>
  clientID: <client-id-of-user-assigned-identity>
  allowedNamespaces:
    list:
    - <cluster-namespace>
```

### Assigning VM identities for cloud provider authentication (self-managed)

When using a user-assigned managed identity to create the workload cluster, a VM identity should also be assigned to each control plane machine in the workload cluster for Azure Cloud Provider to use. See [here](../self-managed/vm-identity.md#managed-identities) for more information.

## User-Assigned Identity Credentials

<aside class="note">

<h1> Note </h1>

This option is only available for 1st party Microsoft applications who have access to the msi data-plane.

</aside>

#### General
This authentication type is similar to user assigned managed identity authentication combined with client certificate
authentication. As a 1st party Microsoft application, one has access to pull a user assigned managed identity's backing
certificate information from the MSI data plane. Using this data, a user can authenticate to Azure Cloud.

#### Prerequisites
A JSON file with information from the user assigned managed identity. It should be in this format:
```json
        {
            "client_id": "0998...",
            "client_secret": "MIIKUA...",
            "client_secret_url": "https://control...",
            "tenant_id": "93b...",
            "object_id": "ae...",
            "resource_id": "/subscriptions/...",
            "authentication_endpoint": "https://login.microsoftonline.com/",
            "mtls_authentication_endpoint": "https://login.microsoftonline.com/",
            "not_before": "2025-02-07T13:29:00Z",
            "not_after": "2025-05-08T13:29:00Z",
            "renew_after": "2025-03-25T13:29:00Z",
            "cannot_renew_after": "2025-08-06T13:29:00Z"
        }
```

Note, the client secret should be a base64 encoded certificate.

The steps to get this information from the MSI data plane are as follows:
1. Make an unauthenticated GET or POST (no Authorization request headers) on the x-ms-identity-url received from ARM to get the token authority and, on older api versions, resource.
2. Get an Access Token from Azure AD using your Resource Provider applicationId and Certificate. The applicationId should match the one you added to your manifest. The response should give you an access token.
3. Perform a GET or POST to MSI on the same URL from earlier to get the Credentials using this bearer token.

#### Creating the AzureClusterIdentity

The corresponding values should be used to create an `AzureClusterIdentity` resource:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: example-identity
  namespace: default
spec:
  type: UserAssignedIdentityCredential
  tenantID: <azure-tenant-id>
  clientID: <client-id-of-user-assigned-identity>
  userAssignedIdentityCredentialsPath: <path-to-JSON-file-with-mi-certifcate-information>
  userAssignedIdentityCredentialsCloudType: "public"
  allowedNamespaces:
    list:
    - <cluster-namespace>
```

## Azure CLI

<aside class="note warning">
<h1> Warning </h1>
CAPZ authenticates by running <code class="hjls">az</code> as the OS user of the controller process. There is no separate client ID or secret. Every cluster in an allowed namespace uses that same Azure login. An empty <code class="hjls">allowedNamespaces</code> object allows every namespace.
</aside>

AzureCLI is an opt-in mode for a local installer or controller process that already has an `az login` session. Do not use it for a persistent in-cluster production controller. For those controllers, use [Workload Identity](./workload-identity.md) (recommended) or a [Service Principal](#service-principal), consistent with [ASO's authentication guidance](https://azure.github.io/azure-service-operator/guide/authentication/). The published CAPZ image does not include Azure CLI. That limits where this mode works, but does not enforce this usage boundary.

The host must already have an `az login` session. The `clientID` property is unused for `AzureCLI` and may be omitted. Every other identity type must include the property, although the API accepts an empty string. `tenantID` is required by the `AzureClusterIdentity` schema. When a subscription is set, CAPZ does not also pass `tenantID` to `az account get-access-token`, because Azure CLI rejects `--tenant` and `--subscription` together. The subscription is `AzureCluster.spec.subscriptionID` or `AzureManagedControlPlane.spec.subscriptionID`. If that field is empty, CAPZ uses `AZURE_SUBSCRIPTION_ID`. If both are empty, CAPZ returns an error and does not create Azure clients.

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: example-identity
  namespace: default
spec:
  type: AzureCLI
  tenantID: <azure-tenant-id>
  allowedNamespaces:
    list:
    - <cluster-namespace>
```

This authenticates only CAPZ. Configure ASO separately. CAPZ does not create a per-cluster ASO Secret for this type.

With no namespace `aso-credential` Secret, CAPZ removes `serviceoperator.azure.com/credential-from` and ASO uses its global credential. CAPZ cannot inspect the global credential's subscription, so it does not verify that it matches the cluster. This is intentional; ASO may target a different subscription than CAPZ.

If an `aso-credential` Secret exists, ASO prefers it over the global credential and does not fall back when that Secret cannot be loaded. It must contain non-empty `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`, and `AZURE_CLIENT_ID`. The subscription must match the cluster spec subscription, or `AZURE_SUBSCRIPTION_ID` when the spec field is empty. CAPZ then removes `credential-from`.

On either of those successful paths, CAPZ deletes a CAPZ-owned leftover Secret. A user-managed Secret with the same name is left in place.

An incomplete Secret stops CAPZ reconciliation and leaves the ASO resource's `credential-from` annotation unchanged. An invalid namespace Secret does not cause ASO to use the global credential. On a subscription mismatch, CAPZ returns an error before changing the ASO resource's `credential-from` or `reconcile-policy` annotations. Ensure the namespace credential and cluster use the same subscription before retrying. If the namespace Secret is removed, CAPZ resumes the documented global-credential path; CAPZ cannot verify that global credential's subscription.

## Azure Host Identity

The identity assigned to the Azure host which in the control plane provides the identity to Azure Cloud Provider, and can be used on all nodes to provide access to Azure services during cloud-init, etc.

- User-assigned Managed Identity
- System-assigned Managed Identity
- Service Principal
- See details about each type in the [VM identity](../self-managed/vm-identity.md) page

More details in [Azure built-in roles documentation](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).
