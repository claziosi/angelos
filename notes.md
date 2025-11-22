# 1. Créer un ConfigMap vide avec l'annotation magique
oc create configmap trusted-ca

# 2. Ajouter l'annotation pour que OpenShift injecte le CA automatiquement
oc annotate configmap trusted-ca service.beta.openshift.io/inject-cabundle=true

# 3. Attendre quelques secondes que OpenShift injecte le CA
sleep 5

# 4. Extraire le CA dans un fichier
oc get configmap trusted-ca -o jsonpath='{.data.service-ca\.crt}' > service-ca.crt
