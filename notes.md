FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MonApplication.csproj", "./"]
RUN dotnet restore "MonApplication.csproj"
COPY . .
RUN dotnet build "MonApplication.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MonApplication.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Créer le répertoire pour les certificats
RUN mkdir -p /certs

# Créer le script de démarrage
RUN echo '#!/bin/bash' > /app/start.sh && \
    echo 'set -e' >> /app/start.sh && \
    echo '' >> /app/start.sh && \
    echo '# Chemins des certificats avec valeurs par défaut' >> /app/start.sh && \
    echo 'CRT_FILE="${CRT_FILE:-/certs/tls.crt}"' >> /app/start.sh && \
    echo 'KEY_FILE="${KEY_FILE:-/certs/tls.key}"' >> /app/start.sh && \
    echo 'PFX_FILE="${PFX_FILE:-/certs/certificate.pfx}"' >> /app/start.sh && \
    echo 'PFX_PASSWORD="${PFX_PASSWORD:-}"' >> /app/start.sh && \
    echo '' >> /app/start.sh && \
    echo '# Vérifier si les fichiers existent' >> /app/start.sh && \
    echo 'if [ -f "$CRT_FILE" ] && [ -f "$KEY_FILE" ]; then' >> /app/start.sh && \
    echo '    echo "Converting CRT and KEY to PFX..."' >> /app/start.sh && \
    echo '    if [ -z "$PFX_PASSWORD" ]; then' >> /app/start.sh && \
    echo '        openssl pkcs12 -export -out "$PFX_FILE" -inkey "$KEY_FILE" -in "$CRT_FILE" -passout pass:' >> /app/start.sh && \
    echo '    else' >> /app/start.sh && \
    echo '        openssl pkcs12 -export -out "$PFX_FILE" -inkey "$KEY_FILE" -in "$CRT_FILE" -passout pass:"$PFX_PASSWORD"' >> /app/start.sh && \
    echo '    fi' >> /app/start.sh && \
    echo '    echo "PFX certificate created at $PFX_FILE"' >> /app/start.sh && \
    echo 'else' >> /app/start.sh && \
    echo '    echo "Warning: Certificate files not found, skipping PFX conversion"' >> /app/start.sh && \
    echo 'fi' >> /app/start.sh && \
    echo '' >> /app/start.sh && \
    echo '# Lancer application .NET' >> /app/start.sh && \
    echo 'exec dotnet MonApplication.dll "$@"' >> /app/start.sh && \
    chmod +x /app/start.sh

ENTRYPOINT ["/app/start.sh"]
CMD ["--urls=https://+:8443", \
     "--Kestrel:Endpoints:Https:Url=https://+:8443", \
     "--Kestrel:Endpoints:Https:Certificate:Path=/certs/certificate.pfx", \
     "--Kestrel:Endpoints:Https:Certificate:Password="]
