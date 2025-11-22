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

# Script de démarrage qui convertit CRT+KEY en PFX
COPY <<'EOF' /app/start.sh
#!/bin/bash
set -e

# Chemins des certificats
CRT_FILE="${CRT_FILE:-/certs/tls.crt}"
KEY_FILE="${KEY_FILE:-/certs/tls.key}"
PFX_FILE="${PFX_FILE:-/certs/certificate.pfx}"
PFX_PASSWORD="${PFX_PASSWORD:-}"

# Vérifier si les fichiers existent
if [ -f "$CRT_FILE" ] && [ -f "$KEY_FILE" ]; then
    echo "Converting CRT and KEY to PFX..."
    
    # Conversion avec OpenSSL (sans mot de passe si PFX_PASSWORD est vide)
    if [ -z "$PFX_PASSWORD" ]; then
        openssl pkcs12 -export -out "$PFX_FILE" \
            -inkey "$KEY_FILE" \
            -in "$CRT_FILE" \
            -passout pass:
    else
        openssl pkcs12 -export -out "$PFX_FILE" \
            -inkey "$KEY_FILE" \
            -in "$CRT_FILE" \
            -passout pass:"$PFX_PASSWORD"
    fi
    
    echo "PFX certificate created at $PFX_FILE"
else
    echo "Warning: Certificate files not found, skipping PFX conversion"
fi

# Lancer l'application .NET
exec dotnet MonApplication.dll "$@"
EOF

RUN chmod +x /app/start.sh

ENTRYPOINT ["/app/start.sh"]
CMD ["--urls=https://+:8443", \
     "--Kestrel:Endpoints:Https:Url=https://+:8443", \
     "--Kestrel:Endpoints:Https:Certificate:Path=/certs/certificate.pfx", \
     "--Kestrel:Endpoints:Https:Certificate:Password="]
