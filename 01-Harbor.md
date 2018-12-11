# Harbor

Harbor is a CNCF project that combines the Docker Registry, Notary and Clair. This gives you image hosting, signing and scanning out of the box. We won't focus on Harbor too much as we have limited time but you can find out more [here](https://github.com/goharbor/harbor/blob/master/README.md). It's likely you may be using a hosted registry service that already supports vulnerability scanning and image signing that builds on top of the same open source software and therefore provides the same functionality. For example, IBM Cloud Container Registry and Azure Container Registry both provide Notary as a Service as part of their Container Registry as a Service offerings. To keep it local and consistent however, we will be using Harbor to provide this functionality.

## Setup
When running a cluster, you would normally add the Harbor CA to your cluster's trusted CAs. Since we are using a local minikube for our cluster, we will just start minikube with making an exception for harbor as an insecure registry. First get your Minikube IP. Don't panic if Minikube takes a minute to start.

```bash
minikube start
minikube ip
minikube stop
```

Then start the cluster with the insecure registry flag.

```bash
minikube start --insecure-registry=https://<Minikube_IP>:30003
```

We will deploy our registry as an application in minikube, using the harbor setup. To install Harbor run:

```bash
kubectl apply -f setup/harbor.yaml
```
This step can take a few minutes. You can check the progress by accessing the application at <Minikube_IP>:30003 in your browser or taking a look at the cluster state.

```bash
kubectl get all
```

This step can take a few minutes. In the meantime, add the certificate to your local machine.

### Add the Harbor CA to your machine's trusted CAs
Adding the certificates to your machine's trusted CAs is not necessary to deploy into the cluster, but we will need it for later steps using Notary.

Retrieve the CA cert from Kubernetes

```bash
kubectl get secrets -o json harbor-harbor-nginx | jq -r '.data["ca.crt"]' | base64 -d > harbor-ca.crt
```

Add trusted root certificate

#### OSX

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain harbor-ca.crt
# Then restart your Docker Daemon for it to take effect.
```

#### Ubuntu/Debian

```bash
sudo cp harbor-ca.crt /usr/local/share/ca-certificates/harbor-ca.crt
sudo update-ca-certificates
```

#### Windows

```bash
certutil -addstore -f "ROOT" harbor-ca.crt
# Then restart your Docker Daemon for it to take effect.
```

Verify that Harbor is now trusted by logging in to the registry.

```bash
docker login -u admin -p kubecon1234 <Minikube_IP>:30003
```

If you get a 400 Bad Requets error, wait a minute and try again.
Head to `<Minikube_IP>:30003/harbor/sign-in` to login to the Harbor UI. Username is `admin`, password is `kubecon1234`. You may need to add a security exception in the browser as the certificate issuer is untrusted.
