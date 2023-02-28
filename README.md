# webmin-under-kubernetes-ingress
Configure the webmin server as an external service and proxy through ingress kubernetes

```bash
# Env
export WEBMIN_PUBLIC_HOSTNAME="webmin-m1.example.com"
export WEBMIN_SERVICE_HOSTNAME="k8s-m1.example.com"

mkdir /home/webmin
cd /home/webmin

# create ingress
tee /home/webmin/ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: default
  name: webmin
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - $WEBMIN_PUBLIC_HOSTNAME
    secretName: webmin-certs
  rules:
  - host: $WEBMIN_PUBLIC_HOSTNAME
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: webmin
            port:
              number: 443
EOF

# create external service, you can change the port if necessary
tee /home/webmin/webmin.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: webmin
  namespace: default
  labels:
    app: webmin
spec:
  type: ExternalName
  externalName: $WEBMIN_SERVICE_HOSTNAME
  ports:
    - name: "https"
      port: 443
      targetPort: 10000
  selector:
    app: webmin
EOF

kubectl apply -f webmin.yaml
kubectl apply -f ingress.yaml

# add hostname to webmin server
tee -a /etc/webmin/miniserv.conf<<EOF
referers=$WEBMIN_PUBLIC_HOSTNAME
redirect_ssl=1
redirect_host=$WEBMIN_PUBLIC_HOSTNAME
EOF

systemctl restart webmin
```

You will also need to configure the Trusted Refferers on a one-time basis in the webmin for WEBMIN_PUBLIC_HOSTNAME or disable verification.

You will probably want to use the user's real IP addresses, so you will have to set the appropriate ingress policy.
```bash
kubectl patch svc ingress-nginx-controller -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

The above example does not require opening the webmin 10000 port for public access, it must be accessible within the cluster network.
