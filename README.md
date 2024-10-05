# Sui Bridge Node
Requires limiting 50 requests/second per unique IP, this will require change in the nginx
```bash
CONFIG="server {
listen 80;
server_name ${SUBDOMAIN};

    # Define a limit request zone that tracks the number of requests from each IP
    limit_req_zone \$binary_remote_addr zone=ip_limit:10m rate=50r/s;

    location / {
      limit_req zone=ip_limit burst=10 nodelay;  # Apply the rate limit here
      proxy_pass http://${IP}:${PORT};
      proxy_set_header Host \$host;
      proxy_set_header X-Real-IP \$remote_addr;
      proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto \$scheme;
    }
}"

echo "${CONFIG}" | sudo tee /etc/nginx/sites-available/${SUBDOMAIN}
```

If runs in two mode, client true, or false, if client is enabled, a Sui Account Key with enough SUI balance is needed as BridgeClient submits transaction on Sui Network.

# Playbook
```bash
ansible-lint --fix
ansible-playbook setup_bridge.yml -i inventory/testnet.yml --limit "sui"
```