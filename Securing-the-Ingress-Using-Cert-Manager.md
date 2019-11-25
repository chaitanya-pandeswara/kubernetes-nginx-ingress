### <ins>Securing the Ingress Using Cert-Manager:</ins>

To secure your Ingress Resources, you’ll install Cert-Manager, create a `clusterissuer` for production, and modify the configuration of your Ingress to take advantage of the TLS certificates. Cluster Issuers are Cert-Manager Resources in Kubernetes that provision TLS certificates. Once installed and configured, your app will be running behind HTTPS.

Before installing Cert-Manager to your cluster via Helm, you’ll manually apply the required [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)(Custom Resource Definitions) from the `jetstack/cert-manager` repository by running the following command:

```
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.11/deploy/manifests/00-crds.yaml
```

You will see the following output:

```
Output

customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
```

This shows that Kubernetes has applied the custom resources you require for `cert-manager`.

Then, create a namespace for `cert-manager` by running the following command:

```
kubectl create namespace cert-manager
```

Next, you’ll need to add the [Jetstack Helm repository](https://charts.jetstack.io/) to Helm, which hosts the Cert-Manager chart. To do this, run the following command:

```
helm repo add jetstack https://charts.jetstack.io
```

Helm will display the following output:

```
Output

"jetstack" has been added to your repositories
```

Finally, install Cert-Manager into the `cert-manager` namespace:

```
helm install --name cert-manager --version v0.11.0 --namespace cert-manager jetstack/cert-manager
```

You will see the following output:

```
Output

NAME:   cert-manager
LAST DEPLOYED: ...
NAMESPACE: cert-manager
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                                    AGE
cert-manager-edit                       4s
cert-manager-view                       4s
cert-manager-webhook:webhook-requester  4s

==> v1/Deployment
NAME                     READY  UP-TO-DATE  AVAILABLE  AGE
cert-manager             0/1    1           0          3s
cert-manager-cainjector  0/1    1           0          3s
cert-manager-webhook     0/1    1           0          3s

==> v1/Pod(related)
NAME                                      READY  STATUS             RESTARTS  AGE
cert-manager-55fff7f85f-bbqb4             0/1    ContainerCreating  0         3s
cert-manager-cainjector-54c4796c5d-5tq5x  0/1    ContainerCreating  0         3s
cert-manager-webhook-77ccf5c8b4-tqf98     0/1    ContainerCreating  0         3s

==> v1/Service
NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
cert-manager          ClusterIP  10.245.11.227  <none>       9402/TCP  4s
cert-manager-webhook  ClusterIP  10.245.209.94  <none>       443/TCP   4s

==> v1/ServiceAccount
NAME                     SECRETS  AGE
cert-manager             1        4s
cert-manager-cainjector  1        4s
cert-manager-webhook     1        4s

==> v1beta1/APIService
NAME                             AGE
v1beta1.webhook.cert-manager.io  3s

==> v1beta1/ClusterRole
NAME                                    AGE
cert-manager-cainjector                 4s
cert-manager-controller-certificates    4s
cert-manager-controller-challenges      4s
cert-manager-controller-clusterissuers  4s
cert-manager-controller-ingress-shim    4s
cert-manager-controller-issuers         4s
cert-manager-controller-orders          4s

==> v1beta1/ClusterRoleBinding
NAME                                    AGE
cert-manager-cainjector                 4s
cert-manager-controller-certificates    4s
cert-manager-controller-challenges      4s
cert-manager-controller-clusterissuers  4s
cert-manager-controller-ingress-shim    4s
cert-manager-controller-issuers         4s
cert-manager-controller-orders          4s
cert-manager-leaderelection             4s
cert-manager-webhook:auth-delegator     4s

==> v1beta1/MutatingWebhookConfiguration
NAME                  AGE
cert-manager-webhook  3s

==> v1beta1/Role
NAME                                    AGE
cert-manager-cainjector:leaderelection  4s
cert-manager:leaderelection             4s

==> v1beta1/RoleBinding
NAME                                                AGE
cert-manager-cainjector:leaderelection              4s
cert-manager-webhook:webhook-authentication-reader  4s
cert-manager:leaderelection                         4s

==> v1beta1/ValidatingWebhookConfiguration
NAME                  AGE
cert-manager-webhook  3s


NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://docs.cert-manager.io/en/latest/reference/issuers.html

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://docs.cert-manager.io/en/latest/reference/ingress-shim.html
```

The output shows that the installation was successful. As listed in the `NOTES` in the output, you’ll need to set up an Issuer to issue TLS certificates.

You’ll now create one that issues Let’s Encrypt certificates, and you’ll store its configuration in a file named `production_issuer.yaml`. Create it and open it for editing:

```
nano production_issuer.yaml
```

Add the following lines:

<ins>production_issuer.yaml</ins>

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Email address used for ACME registration
    email: your_email_address
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Name of a secret used to store the ACME account private key
      name: letsencrypt-prod
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```

This configuration defines a `ClusterIssuer` that contacts Let’s Encrypt in order to issue certificates. You’ll need to replace <span style="color:red">`your_email_address`</span> with your email address in order to receive possible urgent notices regarding the security and expiration of your certificates.

Save and close the file.

Roll it out with `kubectl`:

```
kubectl create -f production_issuer.yaml
```

You will see the following output:

```
Output

clusterissuer.cert-manager.io/letsencrypt-prod created
```

With Cert-Manager installed, you’re ready to introduce the certificates to the Ingress Resource defined in the previous step. Open `hello-kubernetes-ingress.yaml` for editing:

```
nano hello-kubernetes-ingress.yaml
```

Add the highlighted lines:

<ins>hello-kubernetes-ingress.yaml</ins>

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-kubernetes-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - hw1.your_domain
    - hw2.your_domain
    secretName: hello-kubernetes-tls
  rules:
  - host: hw1.your_domain
    http:
      paths:
      - backend:
          serviceName: hello-kubernetes-first
          servicePort: 80
  - host: hw2.your_domain
    http:
      paths:
      - backend:
          serviceName: hello-kubernetes-second
          servicePort: 80
```

The `tls` block under `spec` defines in what Secret the certificates for your sites (listed under `hosts`) will store their certificates, which the `letsencrypt-prod` `ClusterIssuer` issues. This must be different for every Ingress you create.

Remember to replace the `hw1.your_domain` and `hw2.your_domain` with your own domains. When you’ve finished editing, save and close the file.

Re-apply this configuration to your cluster by running the following command:

```
kubectl apply -f hello-kubernetes-ingress.yaml
```

You will see the following output:

```
Output

ingress.extensions/hello-kubernetes-ingress configured
```

You’ll need to wait a few minutes for the Let’s Encrypt servers to issue a certificate for your domains. In the meantime, you can track its progress by inspecting the output of the following command:

```
kubectl describe certificate hello-kubernetes-tls
```

The end of the output will look similar to this:

```
Output

Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  GeneratedKey  51s   cert-manager  Generated a new private key
  Normal  Requested     51s   cert-manager  Created new CertificateRequest resource "hello-kubernetes-tls-1050581645"
  Normal  Issued        25s   cert-manager  Certificate issued successfully
```

When your last line of output reads `Certificate issued successfully`, you can exit by pressing `CTRL + C`. Navigate to one of your domains in your browser to test. You’ll see the padlock to the left of the address bar in your browser, signifying that your connection is secure.

In this step, you have installed Cert-Manager using Helm and created a Let’s Encrypt `ClusterIssuer`. After, you updated your Ingress Resource to take advantage of the Issuer for generating TLS certificates. In the end, you have confirmed that HTTPS works correctly by navigating to one of your domains in your browser.