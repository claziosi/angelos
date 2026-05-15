RUN echo '#!/bin/sh' > /entrypoint.sh && \
    echo 'set -e' >> /entrypoint.sh && \
    echo '[ -f /var/run/secrets/kubernetes.io/serviceaccount/ca.crt ] && export SSL_CERT_FILE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt' >> /entrypoint.sh && \
    echo 'exec dotnet Ubp.Dione.Netcore.ClientData.Front.dll' >> /entrypoint.sh && \
    chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
