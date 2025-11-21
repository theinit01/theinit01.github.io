---
title: "Expose k8s services via Cloudflare Tunnels"
date: 2025-11-21
draft: false
summary: "In this post, I explain how I exposed Kubernetes services from my homelab using Cloudflare Tunnels to bypass CGNAT without port forwarding, VPNs, or a public IP. Instead of using a Cloud VPS with WireGuard, I opted for Cloudflare’s cloudflared daemon running as a DaemonSet inside the cluster for high availability. The blog walks through domain setup using a free Digiplat domain, connecting it to Cloudflare, creating a tunnel, deploying cloudflared on Kubernetes, and publishing internal services like Argo CD to the internet securely. Simple, secure, and zero headache. Staying tuned for securing routes with Cloudflare Access next."
tags:
  - cloudflare
  - kubernetes
  - homelab
  - networking
  - cgnat
---

So, you finally got a homelab but could not access it from outside your network because of CGNAT.  
CGNAT is basically your ISP saying: No public IP for you, share it with 500 strangers instead.  
And then you cry while trying to reach your cluster from the outside.

I faced this too. The initial plan was to use a Cloud VPS and build a WireGuard tunnel, but then I stumbled across Cloudflare Tunnels and instantly liked the simplicity. If I ever face latency issues (as people on Reddit say sometimes happens), I am also considering Pangolin on an Oracle free instance.

Here, I will walk through how I exposed my Kubernetes services through Cloudflare Tunnels. Cloudflare Tunnels uses a lightweight daemon called cloudflared, which we run as a DaemonSet for high availability.

## Cloudflare Domain Setup

Cloudflare Tunnels require a domain. I got a free one from Digiplat:  
https://github.com/DigitalPlatDev/FreeDomain

Shoutout to them for offering free TLDs like:
- .DPDNS.ORG
- .US.KG
- .QZZ.IO
- .XX.KG

After getting your free domain, you need to configure Cloudflare nameservers.

### Steps on the Cloudflare dashboard
1. Create or log in to a Cloudflare account  
2. Go to Websites and click Add a site
3. Enter your Digiplat domain and click Continue
4. Choose the Free plan and Continue
![Image Description](/images/adddns.png)
5. Cloudflare will show you two nameservers
6. Go to your Digiplat account domain settings and replace the existing nameservers with the Cloudflare ones
7. Wait for DNS propagation  
Once completed, the Cloudflare dashboard will show Active next to your domain.

![Image Description](/images/digiplatdns.png)

## Create a Cloudflare Tunnel

1. On Cloudflare dashboard, go to Zero Trust
2. Open Networks
3. Click Tunnels
4. Click Create a tunnel
5. Select Cloudflared
6. Name your tunnel
7. Choose Docker as the deployment method
8. Copy the Token shown

## Deploy cloudflared on Kubernetes

Create the namespace:
```bash
kubectl create ns cloudflared
```

Create the secret:
```bash
kubectl create secret generic cloudflared-token -n cloudflared --from-literal=TUNNEL_TOKEN='<YOUR_TOKEN>'
```

Apply the DaemonSet manifest:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cloudflared
  namespace: cloudflared
  labels:
    app: cloudflared
spec:
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      serviceAccountName: cloudflared
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          imagePullPolicy: IfNotPresent
          args:
            - tunnel
            - --no-autoupdate
            - run
            - --token
            - $(TUNNEL_TOKEN)
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: cloudflared-token
                  key: TUNNEL_TOKEN
          resources:
            requests:
              cpu: 20m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
      restartPolicy: Always
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudflared
  namespace: cloudflared
```
This will deploy a cloudflared instance on all your Kubernetes nodes. In your Cloudflare dashboard, under the tunnel page, you should now see the number of connectors equal to the number of nodes.

### Publish services via Cloudflare

1. In the tunnel page, click Configure
2. Go to Public Hostnames
3. Click Add a public hostname
4. Choose a subdomain like argocd
5. Select your domain
6. Service type: HTTP
7. URL: service DNS name inside the cluster
 ```bash
 argocd-server.argocd.svc.cluster.local:80
 ```

![Image Description](/images/route.png)

Now your app is available at:
```
https://argocd.<yourdomain>
```
Secure access to your cluster without exposing any real ports, without static IPs, and without crying over CGNAT.

### Closing Thoughts

Cloudflare Tunnels are a ridiculously easy way to expose Kubernetes apps securely over the internet. Zero reverse proxy config, no messing with port forwarding, and no public IP needed. If it causes trouble later, I might try Pangolin too.