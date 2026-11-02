# K-Lake on Google Cloud Marketplace

K-Lake indexes the documents an organisation already holds, in place, and
serves them to people and AI assistants with each user's own access rights.
This is the user guide for the K-Lake Kubernetes app on Google Cloud
Marketplace.

## What you get

The app deploys into a namespace of your GKE cluster:

- the K-Lake console and API;
- the indexing workers and the document extraction fleet;
- a PostgreSQL database and a Valkey queue, each on its own persistent disk.

Your documents stay in your systems and the index stays in your cluster.

## Billing

You are billed hourly through your Google Cloud account for the data K-Lake
has indexed:

| Indexed data | USD per GB per month |
| --- | --- |
| Minimum (included up to 2.5 TB) | $375 a month ($4,500 a year) |
| First 50 TB | 0.150 |
| 50 TB to 200 TB | 0.040 |
| 200 TB to 1,000 TB | 0.030 |
| Above 1,000 TB | 0.020 |

Each band applies only to the data inside it: 100 TB is billed as 50 TB at the
first band's price and 50 TB at the second's. Data is counted in whole GB,
rounded up, with 1 TB = 1,000 GB. The deployment size you choose sets the
compute it runs on, not the price.

Commitment-based private offers are available. Contact KDBL.

## Before you deploy

- **A GKE Standard cluster on x86 nodes.** Autopilot is not supported.
- **Capacity:** the small size requests about 6 vCPU and 9 GiB of memory. A
  starting point that also leaves room for more extractors is 3 ×
  n2-standard-8 nodes.
- **vCPU quota:** a new project's default is 24 vCPU across all regions.
- **Zone capacity:** if node creation waits on
  `ZONE_RESOURCE_POOL_EXHAUSTED`, add a node pool in another zone of the same
  region.

## Deploy

1. Open the K-Lake listing in Google Cloud Marketplace and click **Configure**.
2. Choose a cluster and a namespace. Use one namespace per K-Lake deployment.
3. Choose a **deployment size** (small, medium or large) and a **storage
   class** (SSD is recommended).
4. Click **Deploy**. All pods are ready about three minutes later. Some restart
   once or twice while the database comes up; that is expected.

## Open the console

The console is published on an **internal** load balancer: a private IP
reachable from your VPC, peered networks and VPN.

```bash
kubectl -n <namespace> get svc kdbl-ui     # EXTERNAL-IP is the private address
```

From a machine outside the VPC, use a port-forward instead:

```bash
kubectl -n <namespace> port-forward svc/kdbl-ui 8080:80
```

Sign in with the cluster admin token:

```bash
kubectl -n <namespace> get secret kdbl-api-secrets \
  -o jsonpath='{.data.KDBL_CLUSTER_ADMIN_TOKEN}' | base64 -d
```

## Add your first source

In the console, open **Sources → New source** and choose the system to index:
file shares (SMB, NFS), object storage (Google Cloud Storage, S3, Azure Blob),
SharePoint and OneDrive, and others. Adding a source does not start a crawl;
start one from the source's page.

**Google Cloud Storage:** give the source a service-account JSON key with
`roles/storage.objectViewer` on the bucket. (Keyless Workload Identity needs a
Kubernetes service account that the Marketplace deployment does not create;
see the product documentation for a Helm install if you need it.)

## Check what is billed

As a cluster admin, the API shows every hour reported to Google:

```bash
curl -H "Authorization: Bearer <cluster-admin-token>" \
  "https://<console>/api/usage/marketplace?from=2026-11-01&to=2026-11-30"
```

Each row is one billing dimension for one hour, with its status: `accepted`
(billed), `none` (nothing to bill in that band), `pending` (to be reported) or
`expired` / `rejected` (not billed).

## Upgrade

Deploying a newer version over an existing deployment from the Marketplace
page replaces generated credentials. Upgrade with Helm instead, or deploy the
new version to a new namespace and move your sources. Contact KDBL support for
the upgrade procedure for your version.

## Remove

Delete the application from **Kubernetes Engine → Applications**, then delete
the database and queue volumes, which are kept so that your index is not lost
by accident:

```bash
kubectl delete pvc --all -n <namespace>
```

Billing stops when the deployment is removed.

## Support

- Documentation: https://docs.kdbl.com
- Support: support@kdbl.com
- Security issues: security@kdbl.com

## Licence

K-Lake is commercial software from KDBL, used under the terms in
[LICENSE](LICENSE).
