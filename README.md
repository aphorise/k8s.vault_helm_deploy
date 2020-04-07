# Vault Helm Setup on Kubernetes

This is a draft guide to [vault-helm](https://www.vaultproject.io/docs/platform/k8s/helm/) setup & [running](https://www.vaultproject.io/docs/platform/k8s/helm/run/) as well as [its configuration](https://www.vaultproject.io/docs/platform/k8s/helm/configuration/) with some [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) examples.

[![demo](https://asciinema.org/a/317281)](https://asciinema.org/a/317281?autoplay=1)


## Monitoring Kubernetes 
Monitoring the current kubernetes status log or details is possible by way of CLI **`kubectl`** (with tools like `k9s` or just `watch` being helpful) or the [UI kubernetes dashboard](#kubernetes-dashboard-setup).

A separate terminal / shell session would be good to have at hand for the purposes of monitoring which can commonly be achieved with some flag (`--watch` / '-w', `-f`, etc) to `kubectl` where its supported or otherwise via the invoking of `watch ...` interval updates.

```bash
kubectl cluster-info ;
# we have cluster access & confirmed `kubectl cluster-info` is working.
# with gke for example you may need to redo: `gcloud init && cloud container clusters get-credentials CLUSTER_NAME ;`
# with aws: `aws ...`, etc...

kubectl logs vault-0 -f ;  # // follow logs of pod vault-0
kubectl get deployments --all-namespaces --watch ;  # // logs of deployments
kubectl get events --all-namespaces --sort-by=".metadata.creationTimestamp" --watch ; #  // all events
```

An alternate approach that may be easier and useful for familiarisation or quick navigation is the CLI tool [k9s](https://github.com/derailed/k9s) which helps in switching to specific components of kubernetes:

```bash
brew install derailed/k9s/k9s ;
# // for linux: https://github.com/derailed/k9s/releases

k9s ;
# // in k9s press '0' to show all namespaces
# // in k9s CTRL+a allows for quick switch from default pod via
# CTRL+c when done ;
# // you can keep this open to view change in pods or view logs like
```


## kubernetes dashboard setup

The Kubernetes dashboard UI can be installed & configured as demonstrated below.  A Kubernetes dashboard may already be provided to you by way of your providers web console / panel (for example in GKE) and you may not need this step.

```bash
# // for browser UI on 'http://localhost:8001/ui...':
# // install kubernetes dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc7/aio/deploy/recommended.yaml ;

# // ---------------------------------------------------------------------------
# Setup dashboard for cloud managed-kubernetes: EKS, AKS, GKE, etc.
# On GKE we can get UI credentials like:
gcloud config config-helper --format=json | jq -r '.credential.access_token' ;
# // ^^ GKE users will need this token for k8s-dashboard ui login

# // ---------------------------------------------------------------------------
# // DIY Clusters need setup (ditto some cloud providers)
# // get rbac from: https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md#admin-privileges
#kubectl apply -f rbac.yaml ;  # // may also need SA
#kubectl -n kubernetes-dashboard get secret | grep token ; # get INSTANCE_NAME
#kubectl -n kubernetes-dashboard describe secrets kubernetes-dashboard-token-INSTANCE_NAME ;
#kubectl config set-credentials cluster-admin --token='......' ;

# // ---------------------------------------------------------------------------
kubectl proxy ;
# // In Browser go to using token copied or use kubeconfig file:
# http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```

![url: kubernetes dashboard](/zk8s_dashboard.jpg)


## Installation & Configuration 

### Helm Setup 

```bash
# // on macOS:
brew install helm ;
# curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get-helm-3 > get_helm.sh
# chmod 700 get_helm.sh ;
# ./get_helm.sh ;  # Linux
```

### Vault install

```bash
git clone https://github.com/hashicorp/vault-helm.git ;
cd vault-helm ;
helm install vault ./ ;
```

### kubectl status of vault

```bash
kubectl get pods --all-namespaces | grep vault ;
 # NAMESPACE     NAME                                                       READY   STATUS    RESTARTS   AGE
 # default       vault-0                                                    0/1     Running   0          116s
 # default       vault-agent-injector-5b9c8655f9-lm4rh                      1/1     Running   0          117s

kubectl logs vault-0 ;
 # 2020-04-02T11:01:55.353Z [INFO]  core: seal configuration missing, not initialized
 # 2020-04-02T11:01:58.343Z [INFO]  core: seal configuration missing, not initialized
 # ...
 # // this is because we've not updated default helm vaules.yaml

kubectl get deployment --all-namespaces | grep vault ;
 # NAMESPACE     NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
 # default       vault-agent-injector                       1/1     1            1           2m

kubectl get svc --all-namespaces | grep vault ;
 # NAMESPACE     NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
 # default       vault                      ClusterIP   10.97.10.231   <none>        8200/TCP,8201/TCP   2m
 # default       vault-agent-injector-svc   ClusterIP   10.97.9.210    <none>        443/TCP             2m
 # default       vault-internal             ClusterIP   None           <none>        8200/TCP,8201/TCP   2m

kubectl get sa --all-namespaces ;
# // service accounts

kubectl get events  --all-namespaces --sort-by=".metadata.creationTimestamp" ;
# // as above with `--watch ;` # can be helpful 
```


## Vault Configure for Development Mode
Make all needed adjustments to the values.yaml of the vault-helm package.

```bash
# // we want to edit values.yaml to allow for eg: dev
# nano values.yaml ;

helm delete vault ;  # // delete previous helm deployment

# // if you want to remove pv to start anew:
kubectl get pvc --all-namespaces ;

kubectl delete pvc data-vault-0 data-vault-1 data-vault-2 ;  # // ...
# // otherwise you'll have to 'rm -rf vault/data/' from within pods to reset.

helm install vault ./ ;  # // re-install
kubectl logs vault-0 ;  # // should have root tokens now in dev mode & unsealed.
 # ==> Vault server configuration:
 # 
 #              Api Address: http://10.32.1.6:8200
 #                      Cgo: disabled
 #          Cluster Address: https://10.32.1.6:8201
 #               Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
 #                Log Level: info
 #                    Mlock: supported: true, enabled: false
 #            Recovery Mode: false
 #                  Storage: inmem
 #                  Version: Vault v1.3.3
 # 
 # 2020-04-02T12:50:15.778Z [INFO]  proxy environment: http_proxy= https_proxy= no_proxy=
 # 
 # # // ...
 # 
 # The unseal key and root token are displayed below in case you want to
 # seal/unseal the Vault or re-authenticate.
 # 
 # Unseal Key: ......
 # Root Token: root
 # 
 # # // ...
 #

# // for CLI & UI browser access lets port-forward:
kubectl port-forward service/vault 8200:8200 ;  # // redo if redeployed.
# http://127.0.0.1:8200/ui/vault/auth?with=token
```

## Vault Configure for HA Mode with Integrated Raft Storage & UI Enabled

Adjust `values.yaml` to include HA + raft storage.

```bash
# // we want to edit values.yaml
# nano values.yaml ;

helm delete vault ;  # // delete previous helm deployment

# // if you want to remove pv to start anew:
kubectl get pvc --all-namespaces ;

kubectl delete pvc data-vault-0 data-vault-1 data-vault-2 ;  # // ...
# // otherwise you'll have to 'rm -rf vault/data/' from within pods to reset.

helm install vault ./ ;  # // re-install

# // -----------------------------------------------------
# // NODE0 - inline CLI on vault-0 pod:
kubectl exec -it vault-0 -- /bin/vault 'status' ;
# // ... status from pod vault-0

# // NODE0 - shell attach to vault-0 pod:
kubectl exec -it vault-0 env PS1="\u@\h:\w$ " -- /bin/sh ;
# ...
# vault@vault-0:/$ \
vault operator init -key-shares=1 -key-threshold=1 ;  # // VUSEAL1=... populate.
 # Unseal Key 1: B599eB74XhWWTjQN9f2ttIkjZd7RfUQT6Ag+voPgpvg=
 # 
 # Initial Root Token: s.pznxZBwhKyiByMjab2zLFyV3
# ...
# vault@vault-0:/$ \
VUSEAL1='......' && vault operator unseal $VUSEAL1 ;
 # ...
 # Initialized            true
 # Sealed                 false
# ...

# vault@vault-0:/$ \
exit ;  # // or CTRL+D on keyboard


# // -----------------------------------------------------
# // local machine / operator
export VAULT_ADDR='http://0.0.0.0:8200' ;
vault status ;
# // should show the same as earlier kubectl exec version of status.


# // -----------------------------------------------------
# // NODE1 - shell attach to vault-1 pod:
kubectl exec -it vault-1 env PS1="\u@\h:\w$ " -- /bin/sh ;
# vault@vault-1:/$ \
vault operator raft join http://vault-0.vault-internal:8200 ;
 # Key       Value
 # ---       -----
 # Joined    true

# vault@vault-1:/$ \
VUSEAL1='......' && vault operator unseal $VUSEAL1 ;
 # ...
 # Initialized            true
 # Sealed                 false

# vault@vault-1:/$ \
exit ;  # // or CTRL+D on keyboard


# // -----------------------------------------------------
# // NODE2 - shell attach to vault-2 pod:
kubectl exec -it vault-2 env PS1="\u@\h:\w$ " -- /bin/sh ;
# vault@vault-2:/$ \
vault operator raft join http://vault-0.vault-internal:8200 ;
 # Key       Value
 # ---       -----
 # Joined    true

# vault@vault-2:/$ \
VUSEAL1='......' && vault operator unseal $VUSEAL1 ;
 # ...
 # Initialized            true
 # Sealed                 false

# vault@vault-2:/$ \
exit ;  # // or CTRL+D on keyboard


# // -----------------------------------------------------
# // local machine / operator
VTROOT='s.pznxZBwhKyiByMjab2zLFyV3' ;  # // SET your root token from earlier
VAULT_TOKEN=$VTROOT vault operator raft configuration -format=json
# // ...
# // JSON of raft peers currently configured
```


## Reference

 * [GKE Setup](https://github.com/aphorise/gke-setup/)
 * [Get Started with the Google Kubernetes Engine (GKE)](https://docs.bitnami.com/kubernetes/get-started-gke/)

------