# k0s cluster

Steps to reproduce my setup.

- Install first control-plane node

  ```sh
  sudo k0s install controller --enable-worker --no-taints --start
  ```

  - Enroll new nodes
    - On any control-plane node

      ```sh
      sudo k0s token create --role=worker # or --role=controller
      ```

    - On the new node

      ```sh
      echo "<PASTED_OUTPUT_FROM_ABOVE>" > token

      # as worker
      sudo k0s install worker \
        --token-file token --start

      # or as controller
      sudo k0s install controller \
        --enable-worker --no-taints \
        --token-file=token --start
      ```

- Connect dev station (your laptop?) to a control-plane node

  ```sh
  ANY_CONTROL_PLANE_NODE=aws-amd64 \
  ssh $ANY_CONTROL_PLANE_NODE "sudo k0s kubeconfig admin" > ~/.kube/config
  ```

- Ingress: bound to a specific node (I direct all traffic to this node in Cloudflare)

  ```sh
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update

  helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.hostNetwork=true \
    --set controller.service.enabled=false \
    --set controller.extraArgs.default-ssl-certificate=ingress-nginx/wildcard-tls \
    --set controller.config.ssl-redirect="true" \
    --set controller.config.force-ssl-redirect="true" \
    --set controller.nodeSelector."kubernetes\.io/hostname"=aws-amd64
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
    TOKEN='YOUR_CLOUDFLARE_TOKEN' \
    kubectl create secret generic cloudflare-api-token \
      --namespace cert-manager \
      --from-literal=api-token=$TOKEN
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
