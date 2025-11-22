# Set the urls where Kestrel is going to listen
ENV ASPNETCORE_URLS=http://+:80;https://+:443
# location of the certificate file
ENV ASPNETCORE_Kestrel__Certificates__Default__Path=/usr/local/share/ca-certificates/aspnetapp.crt
# location of the certificate key
ENV ASPNETCORE_Kestrel__Certificates__Default__KeyPath=/usr/local/share/ca-certificates/aspnetapp.key
# specify password in order to open certificate key
ENV ASPNETCORE_Kestrel__Certificates__Default__Password=$CERT_PASSWORD
