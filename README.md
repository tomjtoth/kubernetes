# k0s cluster

Common steps for all nodes you're joining to the cluster,
NODE_IP must be forced over wireguard's static IP definitions (maintained via  
[this Ansible role](https://github.com/tomjtoth/ops/tree/main/roles/wireguard)).

```sh
# Download k0s
curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
```

- Install first control-plane node

  ```sh
  mkdir /etc/k0s
  k0s config create > /etc/k0s/k0s.yaml

  export NODE_IP=$(ip addr | grep -Po ''10\.200\.0\.[0-9]+'')
  sed -r 's/((address|peerAddress): ).+/\1'$NODE_IP'/g' -i /etc/k0s/k0s.yaml

  k0s install controller -c /etc/k0s/k0s.yaml \
    --enable-worker --no-taints \
    --kubelet-extra-args="--node-ip=$NODE_IP" \
    --start
  ```

  - Enroll new nodes
    - On any control-plane node

      ```sh
      k0s token create --expiry=1h --role=worker # or --role=controller
      ```

    - On the new node

      ```sh
      echo "<PASTED_OUTPUT_FROM_ABOVE>" > token
      ```

      - as worker

        ```sh
        export NODE_IP=$(ip addr | grep -Po ''10\.200\.0\.[0-9]+'')
        k0s install worker \
          --kubelet-extra-args="--node-ip=$NODE_IP" \
          --token-file token --start
        ```

      - as controller

        ```sh
        mkdir /etc/k0s
        k0s config create > /etc/k0s/k0s.yaml

        export NODE_IP=$(ip addr | grep -Po ''10\.200\.0\.[0-9]+'')
        sed -r 's/((address|peerAddress): ).+/\1'$NODE_IP'/g' -i /etc/k0s/k0s.yaml

        k0s install controller -c /etc/k0s/k0s.yaml \
          --enable-worker --no-taints \
          --kubelet-extra-args="--node-ip=$NODE_IP" \
          --token-file token --start
        ```

The rest is done from your laptop:

- Connect as admin

  ```sh
  ssh ANY_CONTROL_PLANE_NODE "sudo k0s kubeconfig admin" > ~/.kube/config
  ```

- Ingress: bound to a specific node (I direct all traffic to this node in Cloudflare)

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

- TLS certificates
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
      --from-literal=api-token=<YOUR_CLOUDFLARE_TOKEN>
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

- Add monitoring

  ```sh
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
    --namespace prometheus \
    --create-namespace
  ```

- Deploy apps
  - Create secrets if necessary

    ```sh
    alias {é,ö}secrets='f(){
        kubectl create ns $1
        kubectl -n $1 create secret generic $1-secrets --from-env-file=$2
        unset -f f
    }
    f'

    ösecrets namespace path/to/.env
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
      APP_NAMESPACE=saldo \
      PVC_NAME=saldo-pvc \
      PATH_TO_DATA_ON_HOST=/home/ubuntu/saldo-data \
      envsubst < cluster/migrator.yml | kubectl apply -f -
      ```

    - Remove migrator (rerun with `apply` -> `delete` above)
    - Scale back to original replica count
      - or redeploy your app
    - Restore ArgoCD's self-healing/auto-sync (?)
