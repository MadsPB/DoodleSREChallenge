kubectl create configmap keycloak-cache-config -o=yaml --dry-run=client --from-file=kc-ispn.xml > kcconfig.yaml
kubectl apply -f . --recursive