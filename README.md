# Running Gitpod in [Azure AKS](https://azure.microsoft.com/en-gb/services/kubernetes-service/)

Before starting the installation process, you need:

- An Azure account
  - [Create one now by clicking here](https://azure.microsoft.com/en-gb/free/)
- A user account with "Owner" IAM rights on the subscription
- A `.env` file with basic details about the environment.
  - We provide an example of such file [here](.env.example).
- [Docker](https://docs.docker.com/engine/install/) installed on your machine, or better, a Gitpod workspace :)

## Azure authentication

For simplicity, this guide does **not** use an Azure [service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal).
Authentication is done via an interactive URL, similar to this:

```shell
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code ABC123DEF to authenticate.
```

**To start the installation, execute:**

```shell
make install
```

The whole process takes around twenty minutes. In the end, the following resources are created:

- an AKS cluster running Kubernetes v1.20.
- Azure load balancer.
- Azure MySQL database.
- Azure Blob Storage.
- Azure DNS zone.
- Azure container registry.
- [calico](https://docs.projectcalico.org) as CNI and NetworkPolicy implementation.
- [cert-manager](https://cert-manager.io/) for self-signed SSL certificates.
- [Jaeger operator](https://github.com/jaegertracing/helm-charts/tree/main/charts/jaeger-operator) - and Jaeger deployment for Gitpod distributed tracing.
- [gitpod.io](https://github.com/gitpod-io/gitpod) deployment.

### Common errors running make install

- Insufficient regional quota to satisfy request

  Depending on the size of the configured `disks size` and `machine-type`,
  it may be necessary to request an [increase in the service quota](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits)

  *After increasing the quota, retry the installation running `make install`*

- Some pods never start (`Init` state)

  ```shell
  kubectl get pods -l component=proxy
  NAME                     READY   STATUS    RESTARTS   AGE
  proxy-5998488f4c-t8vkh   0/1     Init 0/1  0          5m
  ```

  The most likely reason is because the [DNS01 challenge](https://cert-manager.io/docs/configuration/acme/dns01/) has yet to resolve. If using `SETUP_MANAGED_DNS`, you will need to update your DNS records to point to the Azure DNS zone nameserver.

  Once the DNS record has been updated, you will need to delete all Cert Manager pods to retrigger the certificate request

  ```shell
  kubectl delete pods -n cert-manager --all
  ```

  After a few minutes, you should see the `https-certificate` become ready.

  ```shell
  kubectl get certificate
  NAME                        READY   SECRET                      AGE
  https-certificates          True    https-certificates          5m
  ```

## Verify the installation

First, check that Gitpod components are running.

```shell
kubectl get pods 
NAME                                 READY   STATUS    RESTARTS   AGE
agent-smith-67mj5                    2/2     Running     0          3m24s
agent-smith-khv98                    2/2     Running     0          3m24s
agent-smith-ncvzc                    2/2     Running     0          3m24s
blobserve-85c48c8789-hr486           2/2     Running     0          3m24s
content-service-7786d99476-6z7ws     1/1     Running     0          3m24s
dashboard-679cb8dbf-mm6hg            1/1     Running     0          3m24s
dbinit-session-s5v7f                 0/1     Completed   0          3m23s
image-builder-mk3-6798697948-h994t   2/2     Running     0          3m24s
jaeger-operator-6cc9f79cc8-t5z7p     1/1     Running     0          3m24s
messagebus-0                         1/1     Running     0          3m24s
migrations-j5tcc                     0/1     Completed   0          3m23s
minio-bcb6cdddb-7rgwx                1/1     Running     0          3m23s
minio-bcb6cdddb-nhbqv                1/1     Running     0          3m23s
openvsx-proxy-0                      1/1     Running     0          3m24s
proxy-589657d8d5-p4xwq               2/2     Running     0          3m23s
registry-facade-pks57                2/2     Running     0          3m24s
registry-facade-rwh5l                2/2     Running     0          3m24s
registry-facade-t8jhb                2/2     Running     0          3m25s
server-84ddd9d6b5-fjlpj              2/2     Running     0          3m23s
ws-daemon-95ms7                      2/2     Running     0          3m25s
ws-daemon-psdcv                      2/2     Running     0          3m25s
ws-daemon-q2z2f                      2/2     Running     0          3m25s
ws-manager-bridge-6f775fb4fc-p475k   2/2     Running     0          3m23s
ws-manager-c89cbc75d-bnw9k           1/1     Running     0          3m23s
ws-proxy-757d8f5bf8-7mv2c            1/1     Running     0          3m23s
ws-scheduler-58c65c759-jwtv5         2/2     Running     0          3m24s
```

### Test Gitpod workspaces

When the provisioning and configuration of the cluster is done, the script shows the URL of the load balancer,
like:

Please open the URL `https://<domain>/workspaces`.
It should display the Gitpod login page similar to the next image.

*DNS propagation* can take several minutes.

![Gitpod login page](./images/gitpod-login.png "Gitpod Login Page")

----

## Destroy the cluster and Azure resources

Remove the Azure cluster running:

```shell
make uninstall
```

> The command asks for a confirmation:
> `Are you sure you want to delete: Gitpod (y/n)?`

This will destroy the Kubernetes cluster and allow you to manually delete the cloud storage.
