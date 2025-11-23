# Créer les répertoires nécessaires
mkdir -p /ca-trust/anchors
mkdir -p /ca-trust/bundle

# Copier le CA
cp /ca-source/ca.crt /ca-trust/anchors/openshift-ca.crt

# Copier le bundle CA système existant et ajouter le nôtre
if [ -f /etc/pki/tls/certs/ca-bundle.crt ]; then
  cp /etc/pki/tls/certs/ca-bundle.crt /ca-trust/bundle/ca-bundle.crt
  cat /ca-source/ca.crt >> /ca-trust/bundle/ca-bundle.crt
else
  cp /ca-source/ca.crt /ca-trust/bundle/ca-bundle.crt
fi

echo "✓ CA installé"




  # Copier le CA depuis le ConfigMap
  cp /ca-source/ca.crt /etc/pki/ca-trust/source/anchors/openshift-ca.crt
  
  # Mettre à jour le trust store
  update-ca-trust extract
  
  echo "✓ CA installé avec succès"
