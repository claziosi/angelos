ENTRYPOINT [ "sh", "-c", \
  "[ -f /var/run/secrets/kubernetes.io/serviceaccount/ca.crt ] && export SSL_CERT_FILE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt; exec dotnet Ubp.Dione.Netcore.ClientData.Front.dll" ]
