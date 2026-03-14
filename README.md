# k3s kluster

Tässä konfiguraatio joka palvelee Tukholmassa (kiitos [Oracle](https://www.oracle.com/cloud/free/)).
Aktiiviset sovellukset löytyvät [täältä](https://ttj.hu#apps).

## Initial konfig

Klusterin jokaisessa **agent node**ssa:

```sh
K3S_TOKEN=$(ssh CONTROL_PLANE_NODE "sudo cat /var/lib/rancher/k3s/server/node-token")

curl -sfL https://get.k3s.io | \
    K3S_URL=https://ora-amp-2.subnet03260926.vcn03260926.oraclevcn.com:6443 \
    K3S_TOKEN=$K3S_TOKEN \
    sh -s -
```

**Läppäri**ssä aja tää [`k3s.sh --init-cluster`](./k3s.sh).

## Sovellukset tuotantoon

Tässä lähinnä muistutukset itselleni.

1.  Sovellusten vaatimat salaisuudet
    - avaa hakemisto, josta löytyy `.env.<APP>` tiedosto(t)
    - säädä arvoit oikein
    - sit aja alla komennot

    ```sh
    init_app_secrets(){
        local dotenv
        while [ $# -gt 0 ]; do
            dotenv=.env.$1

            if [ -f $dotenv ]; then
                kubectl create ns $1

                kubectl -n $1 create secret generic $1-secrets \
                    --from-env-file=$dotenv
            else
                echo "  ERROR: '$dotenv' is not a file"
            fi

            shift
        done
    }

    init_app_secrets saldo veripalvelu
    ```

2.  Laita sovellukset pyörimään

    ```sh
    kubectl apply -k .
    ```

Serti kattaa jokaisen subdomain:in `*.ttj.hu`, ja käyttää [Cloudflare:in API-tokenin](https://dash.cloudflare.com/profile/api-tokens).

### TODO

- Saldo data to be backed up to Oracle's object storage
