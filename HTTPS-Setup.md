# HTTPS Setup with Let's Encrypt for BankApp

## 🔒 Overview
This guide shows how to secure your BankApp with HTTPS using cert-manager and Let's Encrypt.

## 📋 Prerequisites
- Domain name (e.g., bankapp.yourdomain.com)
- DNS configured to point to LoadBalancer
- cert-manager installed on Kubernetes
- AWS Load Balancer Controller (for Ingress)

---

## 🚀 Step 1: Install cert-manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Verify installation
kubectl get pods -n cert-manager

# Expected output:
# NAME                                       READY   STATUS    RESTARTS   AGE
# cert-manager-xxxxx                         1/1     Running   0          1m
# cert-manager-cainjector-xxxxx              1/1     Running   0          1m
# cert-manager-webhook-xxxxx                 1/1     Running   0          1m
```

---

## 🔧 Step 2: Create ClusterIssuer for Let's Encrypt

Create file: `letsencrypt-issuer.yml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Let's Encrypt production server
    server: https://acme-v02.api.letsencrypt.org/directory
    
    # Email for certificate expiration notifications
    email: your-email@example.com
    
    # Secret to store ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    
    # HTTP-01 challenge
    solvers:
    - http01:
        ingress:
          class: alb
```

Apply:
```bash
kubectl apply -f letsencrypt-issuer.yml
```

---

## 🌐 Step 3: Create Ingress with TLS

Create file: `bankapp-ingress.yml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bankapp-ingress
  annotations:
    # AWS Load Balancer Controller
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    
    # SSL/TLS
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    
    # cert-manager
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - bankapp.yourdomain.com
    secretName: bankapp-tls
  rules:
  - host: bankapp.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bankapp-service
            port:
              number: 80
```

Apply:
```bash
kubectl apply -f bankapp-ingress.yml
```

---

## 📝 Step 4: Update DNS

1. Get Ingress LoadBalancer URL:
```bash
kubectl get ingress bankapp-ingress
```

2. Create CNAME record in your DNS:
```
bankapp.yourdomain.com  →  CNAME  →  <INGRESS-LB-URL>
```

---

## ✅ Step 5: Verify Certificate

```bash
# Check certificate status
kubectl get certificate

# Expected output:
# NAME          READY   SECRET        AGE
# bankapp-tls   True    bankapp-tls   2m

# Describe certificate
kubectl describe certificate bankapp-tls

# Check secret
kubectl get secret bankapp-tls
```

---

## 🔍 Troubleshooting

### Certificate not ready

```bash
# Check certificate events
kubectl describe certificate bankapp-tls

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Check challenge
kubectl get challenge
kubectl describe challenge <challenge-name>
```

### Common Issues

1. **DNS not propagated**
   - Wait 5-10 minutes for DNS propagation
   - Verify: `nslookup bankapp.yourdomain.com`

2. **HTTP-01 challenge failing**
   - Ensure port 80 is accessible
   - Check security groups allow HTTP traffic

3. **Rate limit exceeded**
   - Let's Encrypt has rate limits
   - Use staging server for testing:
     ```yaml
     server: https://acme-staging-v02.api.letsencrypt.org/directory
     ```

---

## 🔄 Certificate Renewal

cert-manager automatically renews certificates 30 days before expiration.

Check renewal status:
```bash
kubectl get certificate bankapp-tls -o yaml
```

---

## 🧪 Test HTTPS

```bash
# Test with curl
curl -I https://bankapp.yourdomain.com

# Expected:
# HTTP/2 200
# ...

# Check certificate
openssl s_client -connect bankapp.yourdomain.com:443 -servername bankapp.yourdomain.com
```

---

## 📊 Complete Setup Summary

```bash
# 1. Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# 2. Create ClusterIssuer
kubectl apply -f letsencrypt-issuer.yml

# 3. Create Ingress with TLS
kubectl apply -f bankapp-ingress.yml

# 4. Verify
kubectl get certificate
kubectl get ingress

# 5. Access
https://bankapp.yourdomain.com
```

---

## 🔐 Security Best Practices

1. **Use Production Let's Encrypt**
   - Only after testing with staging

2. **Enable SSL Redirect**
   - Force HTTPS for all traffic

3. **Update Security Groups**
   - Allow HTTPS (443) traffic

4. **Monitor Certificate Expiry**
   - Set up alerts for certificate renewal

5. **Use Strong TLS Configuration**
   ```yaml
   alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
   ```

---

## 📝 Alternative: Using AWS Certificate Manager (ACM)

If you prefer AWS ACM instead of Let's Encrypt:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bankapp-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/xxxxx
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: bankapp.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bankapp-service
            port:
              number: 80
```

---

## 🎯 Summary

✅ cert-manager installed  
✅ Let's Encrypt ClusterIssuer configured  
✅ Ingress with TLS created  
✅ DNS configured  
✅ Certificate issued  
✅ HTTPS enabled  
✅ Auto-renewal configured  

**Your BankApp is now secured with HTTPS! 🔒**
