# Securing Kubernetes Clusters with Istio and Auth0

Learn how to secure a Kubernetes cluster (and the applications that run on it) with Istio and Auth0. In the following article, you will start by creating a brand-new cluster, then you will deploy an unsecured sample application and, after testing the deployment, you will learn how to secure this application and its _pods_ with Istio and Auth0.

Keep reading at:

https://auth0.com/blog/securing-kubernetes-clusters-with-istio-and-auth0/

Download Istio

> I like using the latest though...

``` sh
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.14.0 TARGET_ARCH=x86_64 sh -
```

Install Istio

``` sh
istioctl install --set profile=demo -y
```

Setup the App

``` sh
kubectl create ns demo
kubectl label namespace demo istio-injection=enabled
kubectl config set-context $(kubectl config current-context) --namespace=demo
kubectl apply -f platform/kube/bookinfo.yaml
kubectl wait pods --for condition=Ready --timeout -1s --all
kubectl get pods

kubectl apply -f networking/bookinfo-gateway.yaml

kubectl apply -f networking/bookinfo-virtualservice.yaml

# get the cluster ip // it is likely localhost
kubectl get svc -n istio-system -l istio=ingressgateway

# in a separate tab, port forward
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
```

You should see things here: http://localhost:8080/productpage

### Encryption of service-to-service traffic

Istio is "secure by default" merely by injecting the sidecar-proxies alongside the apps — all service to service traffic is authenticated and encrypted.

The control plane mints a certificate for each proxy. You can find it in its configuration.

>Note: To execute the commands below, you need two command-line tools: jq for processing JSON and step for inspecting certificates.

``` sh
istioctl proxy-config all deploy/productpage-v1 -o json | \
  jq -r '.. |."secret"? | select(.name == "default")'
```

The above command prints the certificate used by productpage to mutually authenticate with other workloads within the mesh.

> Note: Istio implements the Secure Production Identity Framework For Everyone (abbr. SPIFFE) to define identity to workloads within the mesh. The SPIFFE specification defines the SPIFFE ID to communicate identity between workloads. Learn more about The SPIFFE Identity and Verifiable Identity Document.

View the certificate here:

``` sh
istioctl proxy-config all deploy/productpage-v1 -o json | \
  jq -r '.. |."secret"?' | \
  jq -r 'select(.name == "default")' | \
  jq -r '.tls_certificate.certificate_chain.inline_bytes' | \
  base64 -d - | step certificate inspect
```

By adopting Istio, all traffic within the mesh is encrypted (using the minted certificates that we printed out earlier). This protects our data from getting sniffed and prevents person-in-the-middle attacks. As a result, even if gaining access to any of the machines or networking devices, attackers won't be able to read the traffic going back and forth.

Additionally, because services mutually authenticate using the issued certificates, you can further improve security by defining the minimum access for each service using AuthorizationPolicies.

Example service-to-service authorization here: https://istio.io/latest/docs/tasks/security/authorization/authz-http/

Validate the Access Tokens

``` sh
kubectl apply -f security/auth0-authn.yaml
```

Add secrets

``` sh
kubectl apply -f security/app-credentials.yaml
```

Patch the productpage to add secrets

``` sh
kubectl -n demo patch deployment productpage-v1 --patch "
spec:
  template:
    spec:
      containers:
      - name: productpage
        image: docker.io/wmanger/productpage-v2
        envFrom:
        - secretRef:
            name: app-credentials
"
```

Make requests to the productpage to see the Bearer

``` sh
kubectl logs deploy/productpage-v1 | grep Bearer | tail -n 1 | \
    awk -F'Bearer ' '{print $2}' | \
    awk -F\\ '{print $1}'
```

Now authorize user using claims from the tokens

``` sh
kubectl apply -f security/policies/
```

Make changes to the productpage and details services.

``` sh
cd ./src/details
docker build . -t wmanger/details-v2
docker push wmanger/details-v2
kubectl rollout restart deployment/details-v1

cd ./src/productpage
docker build . -t wmanger/productpage-v2
docker push wmanger/productpage-v2
kubectl rollout restart deployment/productpage-v1
```

See the Authorization header being passed down to details

> It always is (I need to prove this), but when you set `forwardOriginalToken: true`, the token is in the Authorization header

- [ ] Prove the JWT is ALWAYS passed when you use `RequestAuthentication` with `jwtRules`

``` sh
kubectl logs {details pod}
```

