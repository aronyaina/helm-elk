# Installation
```sh
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

# Verify the operator 
```sh
kubectl logs -n elastic-system sts/elastic-operator

# When login with microk8s skip verification
kubectl logs -n elastic-system sts/elastic-operator --insecure-skip-tls-verify-backend

```

# Install with example 
```sh
helm install es-quickstart elastic/eck-stack -n elastic-stack --create-namespace \
    --values https://raw.githubusercontent.com/elastic/cloud-on-k8s/2.16/deploy/eck-stack/examples/elasticsearch/hot-warm-cold.yaml \
    --values https://raw.githubusercontent.com/elastic/cloud-on-k8s/2.16/deploy/eck-stack/examples/kibana/http-configuration.yaml
```

# Derivating from k8s for self value
```sh
helm install es-quickstart elastic/eck-stack -n elastic-stack --create-namespace \
    --values elastick-values.yml \
    --values kibana-values.yml
```

# Disable k8s tls
```sh
kubectl config set-cluster $(kubectl config current-context) --insecure-skip-tls-verify=true
```

# Verify all is working
```sh
kubectl get elastic -n elastic-stack -l "app.kubernetes.io/instance"=es-quickstart
```
