apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
spec:
  template:
    spec:
      containers:
      - name: mon-conteneur
        image: mon-image
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/tls/private
          readOnly: true
      volumes:
      - name: tls-certs
        secret:
          secretName: mon-service-certs
