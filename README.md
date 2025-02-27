# Production deployment

System checks can help to detect common configuration tasks:

```
python manage.py check --tag email --deploy
```

Checks can be preformed before the web server starts: `python manage.py check --deploy --fail-level=WARNING && DJANGO_DEBUG=0 gunicorn "wsgi:application" --access-logfile - --workers 12 --threads 12 --reload`


## Kubernetes

If you have kubeconfig for your cluster, enable it with
`export KUBECONFIG=./third-space-kubeconfig.yml`

Then list contexts with `kubectl config get-contexts` and choose one with `kubectl config use-context myorg-third-oidc`

To make the context always available, add to ~/.bashrc:
`export KUBECONFIG="/home/z/pproj/elect_hotline/kube/third-kubeconfig.yaml:/home/u1/.kube/config"`

Recommended to install kubie for context and namespace management: `zypper in kubie`

To allow kubie discover your kubeconfig file, copy it to `~/.kube` directory.


### Create namespace

```
kubectl create namespace "ufo-ns"

kubectl config set-context --current --namespace=ufo-ns
```


### Install gateway-api

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.5.1/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.5.1/deploy/nodeport/deploy.yaml
```


### Create secrets

```
read -p "postgres password: " password && kubectl create secret generic postgres-secrets --namespace "ufo-ns" --from-literal POSTGRES_PASSWORD="$password" --from-literal DATABASE_URL="postgresql://pguser:$password@postgres:5432/pgdb"

kubectl create secret generic ufo-secret-key --namespace "ufo-ns" --from-literal DJANGO_SECRET_KEY=`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1`

kubectl delete secret ufo-secrets --ignore-not-found && read -p "GOOGLE_OAUTH2_CLIENT_SECRET: " google_secret && kubectl create secret generic ufo-secrets --namespace "ufo-ns" --from-literal GOOGLE_OAUTH2_CLIENT_SECRET="$google_secret"

kubectl delete secret sendgrid-secrets --ignore-not-found && read -p "SENDGRID_API_KEY: " sendgrid_api_key && kubectl create secret generic sendgrid-secrets --namespace "ufo-ns" --from-literal SENDGRID_API_KEY="$sendgrid_api_key"

```

If you need to print the secret value later, use `kubectl get secrets/postgres-secrets --template={{.data.DATABASE_URL}} | base64 -d`

### Create env configmap

`cp src/env-local.example env-kube`

Edit `env-kube` file to set required environment variables. Then apply:

`kubectl create configmap ufo-config --from-env-file env-kube -o yaml --dry-run='client' | kubectl apply -f -`

If you need to adjust env vars later, you can edit `env-kube` file and then reapply configmap with the above command. Remember to restart deployment pods with `kubectl rollout restart deployment/ufo-deployment` to apply new configmap.

### Apply manifests

```
kubectl apply -f kube/postgres/
kubectl apply -f kube/ufo/
```

### Obtain TLS certificate

Install cert-manager with `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.yaml`

Test with self-signed certificate: `kubectl apply -f kube/tls/test-selfissued.yml`
Test with staging letsencrypt certificate: `kubectl apply -f kube/tls/test-staging.yml`

Finally, obtain production letsencrypt certificate: `kubectl apply -f kube/tls/prod.yml`

### Edit environment variables

`kubectl edit configmaps ufo-config`

### Ingest database fixtures and superuser

`kubectl exec --stdin --tty deployments/ufo-deployment -- /bin/bash`

```
$ ./scripts/regions.py populatedb
$ ./scripts/2020_ankety.py
$ ./manage.py createsuperuser
```


### Port-forwarding for local testing

```
kubectl port-forward services/nginx-gateway 8080:80 --namespace nginx-gateway
```

### Metrics monitoring

#### Prometheus operator

Install Prometheus Operator Custom Resource Definitions

```
kubectl apply --server-side -f https://github.com/prometheus-operator/prometheus-operator/releases/download/v0.80.0/bundle.yaml
```

Prometheus config-reloader has only 10m cpu by default and being constantly throttled. To avoid this, override the args in the prometheus operator deployment: `kubectl apply --server-side -f kube/metrics/prom-operator-deployment.yml`. See https://github.com/prometheus-operator/prometheus-operator/issues/5446#issuecomment-1596664300 and https://github.com/prometheus-operator/kube-prometheus/issues/2333

#### Prometheus

Create prometheus instance: `kubectl apply -f kube/metrics/ufo-prometheus.yaml`

Check prometheus is running. Start port-forwarding: `kubectl port-forward  prometheus-ufo-prometheus-0 9000:9090`. Access it via http://127.0.0.1:9000/.

Note: port 9090 on localhost might be taken by your local kind cluster, so port 9000 is used instead in the forwarding command above.

#### Kubernetes system and node metrics

Create ServiceMonitor rules for kubernetes system metrics:
```
kubectl apply -f kube/metrics/kube-sys/kube-apiserver-servicemon.yml
kubectl apply -f kube/metrics/kube-sys/kubelet-servicemon.yml
```

Create Node exporter and its ServiceMonitor:
```
kubectl apply -f kube/metrics/kube-sys/node-monitor.yml
```

Create alert rules:
```
kubectl apply -f kube/metrics/kube-sys/kube-alert-rules.yml
```

#### Django ServiceMonitor

Create Prometheus ServiceMonitor for ufo django backend: `kubectl apply -f kube/metrics/ufo-servicemon.yml`

Check that ServiceMonitor is up and active. Start port forwarding and go to http://127.0.0.1:9000/targets.

#### Grafana

Install grafana `kubectl apply -k kube/metrics/grafana/`


#### ArgoCD

```
zypper in argocd-cli
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.2/manifests/install.yaml
```

Print admin password: `argocd admin initial-password -n argocd`

Start port forwarding: `kubectl port-forward service/argocd-server -n argocd 8080:443`


### Utils

#### Krew

Allows installation of kubectl plugins

```
(   set -x; cd "$(mktemp -d)" &&   OS="$(uname | tr '[:upper:]' '[:lower:]')" &&   ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&   KREW="krew-${OS}_${ARCH}" &&   curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&   tar zxvf "${KREW}.tar.gz" &&   ./"${KREW}" install krew; )
```

Print all installed plugins with `kubectl krew list`

#### OIDC-login

`kubectl krew install oidc-login`

#### resource-capacity

`kubectl krew install resource-capacity`

Print all resource requests and limits for all pods: `kubectl resource-capacity --pods`

#### Kubectl alias

Add to bashrc alias for kubectl:
```
alias kk="kubectl"
__load_completion kubectl
complete -o default -F __start_kubectl kk
```

##### Kubie

Recommended context and namespace manager tool.

Install: `sudo zypper install fzf kubie`

To allow kubie discover your kubeconfig file, copy it to `~/.kube` directory.


##### Kubectx

Can be used to switch context or namespace globally. It is less convenient than Kubie, which allows to switch context for currently opened shell session.

```
sudo zypper install fzf
kubectl krew install ctx
kubectl krew install ns
```

Now you can switch with `kk ctx` or `kk ns`


####
## Deploy with Vercel / Neondb

```
zypper in postgresql
```

```
DJANGO_SETTINGS_MODULE=settings_neondb ./manage.py migrate --skip-checks
DJANGO_SETTINGS_MODULE=settings_neondb ./scripts/regions.py populatedb
```

### Deploy to Heroku
```
git remote add heroku https://git.heroku.com/spbtest-2019.git
cp env-local.example env-heroku
echo "DJANGO_SECRET_KEY=`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1`" >> env-heroku
export HEROKUAPP=`git remote get-url heroku | python -c "print(input().split('/')[-1][:-4])"`
```

Edit env-heroku file: set DATABASE_URL and other required variables

```
./push_heroku_env.py env-heroku
git push heroku master
heroku run sh -c './manage.py migrate --skip-checks'
heroku run sh -c './scripts/2020_ankety.py'
heroku run sh -c './scripts/regions.py populatedb'
heroku run sh -c './manage.py createsuperuser'
```

### Build and publish docker image

```
docker build --tag fak3/ufo:0.5 .
```

To test, push image into kind-cluster
```
kind load docker-image fak3/ufo:0.5 --name kind-cluster
```

Finally, push to dockerhub
```
docker push fak3/ufo:0.5
```
