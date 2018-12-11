# kubesec plugin and admission controller


## Admission Control

Clone https://github.com/stefanprodan/kubesec-webhook

```bash
$ git clone https://github.com/stefanprodan/kubesec-webhook
$ cd kubesec-webhook
```

Generate the certificates for the webhook, deploy it, and label the `default` namespace so the admission controllers knows to validate its pods:

```bash
$ make certs && make deploy
$ kubectl label namespaces default kubesec-validation=enabled
```

This deploys a `kubesec-webhook` pod in the `kubesec` namespace, a webhook admission controller configuration for the API server, and the secrets for secure communication.

Then examine the webhook admission controller that's just been deployed:

```bash
$ less deploy/webhook-registration.yaml
```

Find the `namespaceSelector` that corresponds to the label applied to the `default` namespace above. This is the only link between the admission controller and a namespace.

Notice a CA bundle is required. This is because pod manifests may contain secrets in environment variables, or sensitive information about itself or other services. To ensure that the Kubernetes API trusts the admission controller, the CA bundle of the HTTPS endpoint that the API server POSTs to must be declared to establish a trust relationship between them.

The secrets that are mounted into the admission controller are in `webhook-certs.yaml`, and the actual pod that's deployed is in `webhook.yaml`.

Try to deploy an insecure deployment:

```bash
$ kubectl apply -f ./test/deployment.yaml
Error from server (InternalError): error when creating "./test/deployment.yaml": Internal error occurred: admission webhook "deployment.admi
ssion.kubesc.io" denied the request: deployment-test score is -30, deployment minimum accepted score is 0
```

The deployment is insecure. Let's check why. Create a Bash function to POST YAML to https://kubesec.io

> all sensitive configuration should live in `secrets` -  never leak configuration to a remote service

```bash
$ kubesec() {
    local FILE="${1:-}";
    [[ ! -f "${FILE}" ]] && {
        echo "kubesec: ${FILE}: No such file" >&2;
        return 1
    };
    curl --silent \
      --compressed \
      --connect-timeout 5 \
      -F file=@"${FILE}" \
      https://kubesec.io/
}
$ kubesec ./test/deployment.yaml
```

The problem is `containers[] .securityContext .privileged == true` - running a privileged pod.

Although this is dangerous, perhaps we have an "urgent business requirement" (:facepalm:). Let's edit the admission controller to allow an insecure deployment:

```bash
$ sed -i 's,-min-score=.*,-min-score=-100,' deploy/webhook.yaml
$ kubectl delete -f deploy/webhook.yaml
$ kubectl create -f deploy/webhook.yaml
```

Now anything with a score above `-100` will be allowed into the cluster! This is a bad thing. Let's test it:

```bash
$ kubectl apply -f ./test/deployment.yaml
deployment.apps "deployment-test" created
```

Now that we've deployed an insecure pod, let's change the admission controller risk threshold back to `0`.

```bash
$ sed -i 's,-min-score=.*,-min-score=0,' deploy/webhook.yaml
$ kubectl delete -f deploy/webhook.yaml
$ kubectl create -f deploy/webhook.yaml
$ kubectl get pods --selector=app=nginx
```

Notice that the existing deployment is not affected - admission controllers are only called when an API call is "admitted" to the API server.

## 1.7.2.2 Kubectl Plugin

Now that the cluster is healthy and preventing risky deployments, use Kubesec
to check the risk of deployments that are already running.

Install the kubectl plugin:

```bash
$ mkdir -p ~/.kube/plugins/scan && \
curl -sL https://github.com/stefanprodan/kubectl-kubesec/releases/download/0.2.0/kubectl-kubesec_0.2.0_`uname -s`_amd64.tar.gz | tar xzvf - -C ~/.kube/plugins/scan
```

This has installed a binary to `~/.kube/plugins/scan`, and added a new command to `kubectl`:

```bash
$ kubectl plugin scan --help
Scan and score Kubernetes resources using security features powered by Kubesec.io

Usage:
  kubectl plugin scan [flags] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```

Choose a pod in the `kube-system` namespace and scan it:

```bash
$ kubectl -n kube-system plugin scan pod/POD_NAME
```

What do these responses mean? How can we use them to enhance the security of our YAML?

<!-- warning and advisories using selectors on YAML. Take heed of them! -->

[https://kubesec.io/](https://kubesec.io/) hosts human-readable rationales for
the risk scores associated with each entry.

What's the most dangerous risk identified?

<!-- privileged, mounted docker socket, host namespaces -->

How can we increase the score of our deployments?

<!-- add security features! Caps, Seccomp, securityContext entries -->

## 1.7.2.3 Add kubesec to the pipeline

This stage checks the risk score of the `deployment.yaml`.

For larger deployments and GitOps repos this can be modified to check only
files that have changed, or to discover and verify all Kubernetes YAML using
`kubesec` and `GNU Parallel`.

```
  stage('Kubesec') {
    sh """
      echo 'Running Kubesec...'
      set -o pipefail

      if curl --silent \
           --compressed \
           --connect-timeout 5 \
           -F file=@deployment.yaml \
           https://kubesec.io/ \
         | jq --exit-status '.score > 10' >/dev/null; then

        exit 0;
      fi

      echo 'Application failed kubesec scan'
      exit 1
    """'
  }
```

> Due to the way Jenkins parses shell blocks for variable substitutions, we
> were unable to write the above code more nicely, e.g. to use the `kubesec`
> script from earlier.

Why bother with the admission controller if we have this step?

<!--
- because it can be circumvented by pushing anyway, either from jenkins or
  manually
-->
