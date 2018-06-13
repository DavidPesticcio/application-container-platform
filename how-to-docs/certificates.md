### **Application Certificates**
-----

The platform provide's an in-house certificate authority to provide local hostname.namespace.svc.cluster.local certificates. Note: the CA is setup only to serve certificates under hostname.namespace.svc.clutser.local, ip addresses and single named hostnames, i.e. localhost, myservice etc.

#### **CA Bundle**

By default, in the all namespaces a CA bundle has been added which can been mounted into the /etc/ssl/certs of the container and which contains the root CA used to verify authenticity of the certificates. An example of using it is given below.

#### **Example Deployment**

Although technically the certificates aren't exclusively for ingress, the following show's a common deployment of external service via [ingress](https://github.com/UKHomeOffice/application-container-platform/blob/master/how-to-docs/ingress.md) with backend services / pods using tls between ingress controller and themselves.

First we have to acquire the certficate from the CA.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: example
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: example
    spec:
      containers:
      - name: certs
        # PLEASE do not use latest, but check for the latest tag in the releases page of https://github.com/UKHomeOffice/cfssl-sidekick
        image: quay.io/ukhomeofficedigital/cfssl-sidekick:latest
        securityContext:
          runAsNonRoot: true
        args:
        - --certs=/certs
        - --domain=myservice.${KUBE_NAMESPACE}.svc.cluster.local
        - --domain=another_domain_name
        - --expiry=8760h
        env:
        - name: KUBE_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        # an emptyDir which the sidekicks writes the certificates
        - name: certs
          mountPath: /certs
        # The platform CA Bundle hold the root ca used to verify the certificate chain
        - name: bundle
          mountPath: /etc/ssl/certs
          readOnly: true
      - name: your_application
        image: quay.io/ukhomeofficedigital/some_image
        ...
        ports:
        - name: https
          port: 443
          targetPort: 443
        volumeMounts:
        # You can configure your application to pick up the certificates from here (tls.pem and tls-key.pem)
        - name: certs
          mountPath: /certs
          readOnly: true
      # Ensure you mount the volumes
      volumes:
      - name: bundle
        configMap:
          name: bundle
      - name: certs
        emptyDir: {}
```

The cfssl sidekick will request the certificates and place into the /certs directory which is shared between app and itself. Note, the ```--expiry`` can be used to configure the rotation of the certificates, although it's presently up to the application to reload them.


Next we deploy the service.

```shell
apiVersion: v1
kind: Service
metadata:
  name: myservice
  labels:
    name: myservice
spec:
  type: ClusterIP
  ports:
  - name: https
    port: 443
    targetPort: 443
  selector:
    name: myservice
```

We then connect the services together, tying up the ingress and backend services.

```shell
kind: Ingress
metadata:
  name: ingress
  annotations:
    # this tells the ingress to use tls between itself and service
    ingress.kubernetes.io/secure-backends: "true"
    # this selects the external ingress - internet facing
    kubernetes.io/ingress.class: "nginx-external"
  labels:
    # this asks for a certificate from letsencrypt
    stable.k8s.psg.io/kcm.class: default
spec:
  tls:
  - hosts:
    - hostname.example.gov.uk
    secretName: tls
  rules:
  - host: hostname.example.gov.uk
    http:
      paths:
      - path: /
        backend:
          serviceName: myservicename
          servicePort: 443
```

Adding the network policy. Note, by default we place a default deny into everyone namespace, blocking all traffics between pods. The following adds a network policy permitting the ingress to speak to the pods.

```shell
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: permit-traffic
spec:
  podSelector:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-external
    ports:
    - protocol: TCP
      port: 443
```

**HTTP Challenge**

The above example requires that the DNS zone of the ingress hostname you are adding is located in the AWS account that kube-cert-manager resides, i.e. it has the ability to update records on your behalf. Assuming this is not the case you will need to use a HTTP challenge rather than the default DNS one. You will need to change your DNS hostname to a CNAME of ingress-external / ingress-internal ENVIRONMENT.acp.homeoffice.gov.uk i.e assuming notprod environment mysite.domain.com -> ingress-external.notprod.acp.homeoffice.gov.uk

#### **Note: if you are placing your ingress behind the VPN i.e. using the ingress-class `ingress-internal`, the HTTP challenge will not work, please contact the ACP team as you will need to move or delegate your domain to Route53**


```shell
kind: Ingress
metadata:
  name: ingress
  annotations:
    # this tells the ingress to use tls between itself and service
    ingress.kubernetes.io/secure-backends: "true"
    # this selects the external ingress - internet facing
    kubernetes.io/ingress.class: "nginx-external"
    # change the provider to http
    stable.k8s.psg.io/kcm.provider: http
  labels:
    # this asks for a certificate from letsencrypt
    stable.k8s.psg.io/kcm.class: default
spec:
  tls:
  - hosts:
    - hostname.example.gov.uk
    secretName: hostname_tls
  rules:
  - host: hostname.example.gov.uk
    http:
      paths:
      - path: /
        backend:
          serviceName: myservicename
          servicePort: 443
```

**LetsEncrypt Limits**

Note Letscrypt while a free service does come with a number of service limits detailed [here](https://letsencrypt.org/docs/rate-limits/). Probably one of the most crucial for projects is the max certificate requests per week; currently standing at 20. In addition, there is a max 5 failures for per hostname with a freeze of 1 hour, so if you accidently mess up configuration you might hit this.
