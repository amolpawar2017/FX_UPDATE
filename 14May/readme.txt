# Edit locally on the host first, then push it into the container
docker cp C:/Amol/FX/fx-transaction-processor/frontend/src/assets/syntax-config.json \
  fx-frontend:/usr/share/nginx/html/assets/syntax-config.json
