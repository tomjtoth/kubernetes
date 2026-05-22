# Multi-cloud k0s cluster

Using a Wireguard mesh (maintained via [this Ansible role](https://github.com/tomjtoth/ops/tree/main/roles/wireguard)), you can bring all nodes to the same subnet, reducing the amount of ports needed to be open on firewalls.
This setup uses a Virtual IP, keepalived and related scripts on controllers to update VIP owner on all Wireguard peers.

## Common steps

Paste the below function into your shell on the node you want to join.

```sh
_join() {

  # Download k0s
  if [ ! -e /usr/local/bin/k0s ]; then
    curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
  fi

  local join_flags tok=/tmp/k0s-join-token \
    node_ip=$(ip addr | grep -Po '10.200.0.[0-9]+' | head -n 1) \
    usage="
    usage: _join ROLE [TOKEN]
    where:
      ROLE        'worker' or 'controller'
      TOKEN       generated output of an existing controller
    "

  if [ $# -lt 1 ]; then
    echo "$usage"
    return 1
  fi

  # parse role
  case "$1" in
    worker)
      join_flags+=(worker)
    ;;

    controller)
      sudo mkdir /etc/k0s 2>/dev/null
      k0s config create | sudo tee /etc/k0s/k0s.yaml >/dev/null

      sudo yq -yi \
        --arg nodeIp $node_ip \
        '
        .spec.api.address = $nodeIp |
        .spec.storage.etcd.peerAddress = $nodeIp |
        .spec.network.provider = "calico" |
        .spec.network.calico.mode = "bird" |
        .spec.network.nodeLocalLoadBalancing.enabled = true |
        .spec.telemetry.enabled = true
        ' /etc/k0s/k0s.yaml

      join_flags+=(
        controller -c /etc/k0s/k0s.yaml
        --enable-worker --no-taints
      )
    ;;

    -h|--help)
      echo "$usage"
      return 0
    ;;

    *)
      echo "$usage"
      return 1
    ;;
  esac


  if [ -n "$2" ]; then
    echo "$2" > $tok
    join_flags+=(--token-file $tok)
  fi


  sudo k0s install ${join_flags[@]} \
    --kubelet-extra-args="--node-ip=$node_ip" \
    --start
}
```

- Install first control-plane node

  ```sh
  _join controller
  ```

- Enroll new nodes
  - On any control-plane node

    ```sh
    k0s token create --expiry=1h --role=worker

    # OR

    k0s token create --expiry=1h --role=controller
    ```

  - on the node to join

    ```sh
    _join worker "PASTED_OUTPUT_FROM_ABOVE"

    # OR

    _join controller "PASTED_OUTPUT_FROM_ABOVE"
    ```

## Attaching to cluster as admin

Commands from this point forward are executed on your workstation (laptop?).

```sh
ssh ANY_CONTROL_PLANE_NODE "sudo k0s kubeconfig admin" > ~/.kube/config
```

### Installing charts

- Ingress

  ```sh
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update

  helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.kind=DaemonSet \
    --set controller.hostNetwork=true \
    --set controller.service.enabled=false \
    --set controller.extraArgs.default-ssl-certificate=ingress-nginx/wildcard-tls \
    --set controller.config.ssl-redirect="true" \
    --set controller.config.force-ssl-redirect="true"
  ```

- Wildcard TLS certificates for the whole cluster
  - Add cert-manager

    ```sh
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --set crds.enabled=true
    ```

  - Provide Cloudflare [API token](https://dash.cloudflare.com/profile/api-tokens)
    to cert-manager

    ```sh
    kubectl create secret generic cloudflare-api-token \
      --namespace cert-manager \
      --from-literal=api-token="YOUR_CLOUDFLARE_API_TOKEN"
    ```

  - Adjust the specified `dnsNames` for the _wildcard-tls_ [here](./cluster/cert-manager.yml),
    then:

    ```sh
    kubectl apply -k cluster/
    ```

- Distributed storage

  ```sh
  helm repo add longhorn https://charts.longhorn.io
  helm repo update
  helm install longhorn longhorn/longhorn \
    --namespace longhorn-system \
    --create-namespace
  ```

- Monitoring

  ```sh
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
    --namespace prometheus \
    --create-namespace
  ```

- GitOps

  ```sh
  helm repo add argo https://argoproj.github.io/argo-helm
  helm repo update
  helm install argocd argo/argo-cd \
    --namespace argocd \
    --create-namespace
  ```

- Deploy apps
  - Import your app's secrets

    ```sh
    _app(){
        kubectl create ns $1
        kubectl -n $1 create secret generic $1-secrets --from-env-file=$2
    }

    _app namespace path/to/.env
    ```

  - Use ArgoCD or FluxCD for GitOps

  - Or deploy manually

    ```sh
    kubectl apply -k path/to/kustomization
    ```

  - Migrate data if necessary
    - Disable ArgoCD's self-healing/auto-sync (?)

    - Scale down the running app

      ```sh
      kubectl scale deployment saldo-dep -n saldo --replicas=0
      ```

    - Create migrator in the app's namespace

      ```sh
      APP_HOST=azu-2 \
      APP_NAMESPACE=saldo \
      PVC_NAME=saldo-pvc \
      PATH_TO_DATA_ON_HOST=/home/ubuntu/saldo-data \
      envsubst < cluster/migrator.yml | kubectl apply -f -
      ```

    - Remove migrator (rerun the above command, only replace `apply` -> `delete`)
    - Scale back to original replica count
      - or redeploy your app
    - Restore ArgoCD's self-healing/auto-sync (?)
