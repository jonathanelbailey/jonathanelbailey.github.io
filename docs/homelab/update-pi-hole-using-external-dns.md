# Update Pi-Hole DNS Records With External DNS

## Create Pi-Hole Secret

```shel
kubectl create ns external-dns
create secret generic pihole-secret -n external-dns --from-literal=EXTERNAL_DNS_PIHOLE_PASSWORD=<PASSWORD>
```

## Deploy External-DNS
