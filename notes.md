---
# NetworkPolicy 1: Autoriser le trafic entrant depuis d'autres namespaces
# À déployer dans le namespace où se trouvent vos pods API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-from-other-namespaces
  namespace: votre-namespace-api  # MODIFIER: namespace contenant votre API
spec:
  podSelector:
    matchLabels:
      app: votre-api  # MODIFIER: label de vos pods API
  policyTypes:
    - Ingress
  ingress:
    # Règle 1: Autoriser depuis un namespace spécifique
    - from:
        - namespaceSelector:
            matchLabels:
              name: namespace-client  # MODIFIER: label du namespace client
      ports:
        - protocol: TCP
          port: 8443  # Port d'écoute du pod
    
    # Règle 2 (optionnel): Autoriser depuis TOUS les namespaces
    # Décommenter si vous voulez autoriser n'importe quel namespace
    # - from:
    #     - namespaceSelector: {}
    #   ports:
    #     - protocol: TCP
    #       port: 8443

---
# NetworkPolicy 2: Autoriser la communication pod-à-pod dans le même namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-pod-to-pod-same-namespace
  namespace: votre-namespace-api  # MODIFIER: namespace contenant votre API
spec:
  podSelector:
    matchLabels:
      app: votre-api  # MODIFIER: label de vos pods API
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: votre-api  # Même label = pods du même déploiement
      ports:
        - protocol: TCP
          port: 8443

---
# NetworkPolicy 3: Autoriser le trafic sortant (Egress)
# Nécessaire si vous avez des politiques Egress restrictives
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress
  namespace: votre-namespace-api  # MODIFIER: namespace contenant votre API
spec:
  podSelector:
    matchLabels:
      app: votre-api  # MODIFIER: label de vos pods API
  policyTypes:
    - Egress
  egress:
    # Autoriser vers les pods du même namespace
    - to:
        - podSelector:
            matchLabels:
              app: votre-api
      ports:
        - protocol: TCP
          port: 8443
    
    # Autoriser le DNS (essentiel pour la résolution de noms de service)
    - to:
        - namespaceSelector:
            matchLabels:
              name: openshift-dns
      ports:
        - protocol: UDP
          port: 53
    
    # Autoriser vers d'autres services si nécessaire (bases de données, etc.)
    - to:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 443
        - protocol: TCP
          port: 5432  # Exemple: PostgreSQL

---
# IMPORTANT: Labéliser vos namespaces pour les sélecteurs
# Commande à exécuter pour labéliser votre namespace client:
# oc label namespace namespace-client name=namespace-client

# Commande pour labéliser le namespace de l'API (si nécessaire):
# oc label namespace votre-namespace-api name=votre-namespace-api
