```bash
kubectl -n kube-system create secret generic hcloud \
--from-literal=token=<YOUR_HETZNER_API_TOKEN> \
--from-literal=network=<YOUR_HETZNER_NETWORK_ID>

helm repo add hcloud https://charts.hetzner.cloud
helm repo update hcloud
helm install hccm hcloud/hcloud-cloud-controller-manager -n kube-system

# Use IPv4 DNS
#DNS Policy options:
#
#Default - Uses the node's /etc/resolv.conf
#ClusterFirst - Uses CoreDNS (default for pods)
#None - What you used before with manual nameservers

kubectl patch deployment -n kube-system hcloud-cloud-controller-manager --patch '
{
  "spec": {
    "template": {
      "spec": {
        "dnsPolicy": "None",
        "dnsConfig": {
          "nameservers": ["8.8.8.8", "8.8.4.4"],
          "searches": ["default.svc.cluster.local", "svc.cluster.local", "cluster.local"],
          "options": [
            {"name": "ndots", "value": "2"},
            {"name": "edns0"},
            {"name": "timeout", "value": "5"}
          ]
        }
      }
    }
  }
}'


kubectl patch deployment -n kube-system hcloud-cloud-controller-manager --patch '
{
  "spec": {
    "template": {
      "spec": {
        "dnsPolicy": "ClusterFirst"
      }
    }
  }
}'

kubectl patch deployment -n kube-system hcloud-cloud-controller-manager --patch '
{
 "spec": {
   "template": {
     "spec": {
       "containers": [
         {
           "name": "hcloud-cloud-controller-manager",
           "env": [
             {
               "name": "HCLOUD_NETWORK",
               "valueFrom": {
                 "secretKeyRef": {
                   "name": "hcloud",
                   "key": "network"
                 }
               }
             }
           ]
         }
       ]
     }
   }
 }
}'

```
