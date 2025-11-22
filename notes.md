apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-dotnet
spec:
  template:
    spec:
      initContainers:
      - name: cert-converter
        image: alpine/openssl
        command: ["/bin/sh", "-c"]
        args:
          - |
            openssl pkcs12 -export \
              -out /certs-pfx/certificate.pfx \
              -inkey /certs/tls.key \
              -in /certs/tls.crt \
              -passout pass:
        volumeMounts:
        - name: tls-certs
          mountPath: /certs
          readOnly: true
        - name: pfx-cert
          mountPath: /certs-pfx
      containers:
      - name: dotnet-app
        image: mon-image-dotnet:latest
        volumeMounts:
        - name: pfx-cert
          mountPath: /app/certs
          readOnly: true
      volumes:
      - name: tls-certs
        secret:
          secretName: mon-service-certs
      - name: pfx-cert
        emptyDir: {}
