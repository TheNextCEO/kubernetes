## Create three virtual machines

multipass launch -c 1 -m 1G -d 16G -n k3s-master 18.04 --cpu-architecture x86_64

for f in 1 2; do
multipass launch -c 1 -m 1G -d 16G -n k3s-worker-$f 18.04 --cpu-architecture x86_64
done

multipass list

## Deploy K3s

multipass exec k3s-master -- bash -c "curl -sfL https://get.k3s.io | sh -"

TOKEN=$(multipass exec k3s-master sudo cat /var/lib/rancher/k3s/server/node-token)

IP=$(multipass info k3s-master | grep IPv4 | awk '{print $2}')

for f in 1 2; do
multipass exec k3s-worker-$f -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=\"https://$IP:6443\" K3S_TOKEN=\"$TOKEN\" sh -"
done

## Get the Cluster Configuration

multipass exec k3s-master sudo cat /etc/rancher/k3s/k3s.yaml > k3s.yaml

sed -i '' "s/127.0.0.1/$IP/" k3s.yaml

export KUBECONFIG=$PWD/k3s.yaml

## Deploy ArgoCD

kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl port-forward svc/argocd-server -n argocd 8080:443

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Final step: clean-up everything

multipass stop k3s-master k3s-worker-1 k3s-worker-2
multipass delete k3s-master k3s-worker-1 k3s-worker-2
multipass purge
