# Geth and Lighthouse Node Setup on Ubuntu

This guide will walk you through setting up a Geth full node and a Lighthouse consensus client on Ubuntu, including configuring the firewall, setting up services, and exposing RPC endpoints with Nginx.

## Table of Contents

- [1. Configuring UFW Firewall for Geth](#1-configuring-ufw-firewall-for-geth)
- [2. Installing Geth Execution Client](#2-installing-geth-execution-client)
- [3. Installing Lighthouse Consensus Client](#3-installing-lighthouse-consensus-client)
- [4. Testing Geth RPC Endpoints](#4-testing-geth-rpc-endpoints)
- [5. Setting Up Nginx with HTTPS and Basic Authentication](#5-setting-up-nginx-with-https-and-basic-authentication)
- [6. Exposing WebSocket RPC Endpoint](#6-exposing-websocket-rpc-endpoint)

## 1. Configuring UFW Firewall for Geth

1. **Update and Install Required Packages:**

    ```bash
    sudo apt-get update
    sudo apt-get install ufw net-tools
    ```

2. **Configure UFW Firewall:**

    ```bash
    sudo ufw disable
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 22/tcp      # SSH access
    sudo ufw allow 30303       # Geth P2P
    sudo ufw allow 9000        # Lighthouse P2P
    sudo ufw allow 80          # HTTP
    sudo ufw allow 443         # HTTPS
    sudo ufw enable
    sudo ufw status verbose
    ```

## 2. Installing Geth Execution Client

1. **Add Ethereum PPA and Install Geth:**

    ```bash
    sudo add-apt-repository ppa:ethereum/ethereum
    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get install ethereum
    ```

2. **Create Geth Service File:**

    Create `/lib/systemd/system/geth.service` with the following content:

    ```ini
    [Unit]
    Description=Geth Full Node
    After=network-online.target
    Wants=network-online.target

    [Service]
    WorkingDirectory=/home/ubuntu
    User=ubuntu
    ExecStart=/usr/bin/geth --http --http.addr "0.0.0.0" --http.port "8545" --http.corsdomain "*" --http.api personal,eth,net,web3,debug,txpool,admin --authrpc.jwtsecret /tmp/jwtsecret --ws --ws.port 8546 --ws.api eth,net,web3,txpool,debug --ws.origins="*" --metrics --maxpeers 150 
    Restart=always
    RestartSec=5s

    [Install]
    WantedBy=multi-user.target
    ```

3. **Enable and Start Geth Service:**

    ```bash
    sudo systemctl enable geth
    sudo systemctl start geth
    sudo journalctl -f -u geth
    ```

## 3. Installing Lighthouse Consensus Client

1. **Download and Install Lighthouse:**

    ```bash
    wget https://github.com/sigp/lighthouse/releases/download/v4.5.0/lighthouse-v4.5.0-x86_64-unknown-linux-gnu.tar.gz
    tar zxvf lighthouse-v4.5.0-x86_64-unknown-linux-gnu.tar.gz
    sudo mv lighthouse /usr/local/bin/
    ```

2. **Create Lighthouse Service File:**

    Create `/lib/systemd/system/lighthouse.service` with the following content:

    ```ini
    [Unit]
    Description=Lighthouse consensus client
    After=network-online.target
    Wants=network-online.target

    [Service]
    WorkingDirectory=/home/ubuntu
    User=ubuntu
    ExecStart=/usr/local/bin/lighthouse bn --network mainnet --execution-endpoint http://localhost:8551 --execution-jwt /tmp/jwtsecret --checkpoint-sync-url https://mainnet.checkpoint.sigp.io --disable-deposit-contract-sync
    Restart=always
    RestartSec=5s

    [Install]
    WantedBy=multi-user.target
    ```

3. **Enable and Start Lighthouse Service:**

    ```bash
    sudo systemctl enable lighthouse
    sudo systemctl start lighthouse
    ```

## 4. Testing Geth RPC Endpoints

1. **Attach to Geth Console:**

    ```bash
    geth attach
    ```

2. **Check Peers and Sync Status:**

    ```js
    admin.peers.map((el) => el.network.inbound)
    eth.blockNumber
    eth.syncing
    ```

3. **Check Geth Logs:**

    ```bash
    sudo journalctl -f -u geth
    ```

4. **Query Block Number via HTTP RPC:**

    ```bash
    curl -X POST http://127.0.0.1:8545 \
    -H "Content-Type: application/json" \
    --data '{"jsonrpc":"2.0", "method":"eth_blockNumber", "id":1}'
    ```

## 5. Setting Up Nginx with HTTPS and Basic Authentication

1. **Install Nginx and Certbot:**

    ```bash
    sudo apt-get install nginx apache2-utils
    sudo apt-get install python3-certbot-nginx
    ```

2. **Obtain SSL Certificates:**

    ```bash
    sudo certbot --nginx -d example.com
    @monthly root certbot -q renew
    ```

3. **Create Basic Authentication File:**

    ```bash
    sudo htpasswd -c /etc/nginx/htpasswd.users your_user
    ```

4. **Configure Nginx:**

    Create or update `/etc/nginx/sites-available/default` with the following configuration:

    ```nginx
    server {
        server_name example.com;

        location / {
            if ($request_method = OPTIONS) {
                add_header Content-Length 0;
                add_header Content-Type text/plain;
                add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
                add_header Access-Control-Allow-Origin $http_origin;
                add_header Access-Control-Allow-Headers "Authorization, Content-Type";
                add_header Access-Control-Allow-Credentials true;
                return 200;
            }

            auth_basic "Restricted Access";
            auth_basic_user_file /etc/nginx/htpasswd.users;
            proxy_pass http://localhost:8545;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_cache_bypass $http_upgrade;
        }

        listen [::]:443 ssl ipv6only=on;
        listen 443 ssl;
        ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    }

    server {
      if ($host = example.com) {
          return 301 https://$host$request_uri;
      }

      listen 80;
      listen [::]:80;
      server_name example.com;
      return 404;
    }
    ```

5. **Test and Restart Nginx:**

    ```bash
    sudo nginx -t
    sudo service nginx restart
    ```

6. **Query Geth via HTTPS with Authentication:**

    ```bash
    curl -X POST https://example.com \
      -H "Content-Type: application/json" \
      -u your_user:your_password \
      --data '{"jsonrpc":"2.0", "method":"eth_blockNumber", "id":1}'
    ```

## 6. Exposing WebSocket RPC Endpoint

1. **Add WebSocket Location to Nginx Configuration:**

    Update `/etc/nginx/sites-available/default` with:

    ```nginx
    location /ws {
        proxy_http_version 1.1;
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/htpasswd.users;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_pass  http://127.0.0.1:8546/;
    }
    ```

2. **Restart Nginx:**

    ```bash
    sudo service nginx restart
    ```

---

This README should provide clear and structured instructions for setting up your Geth and Lighthouse nodes on Ubuntu. If you have any additional sections or specific details to include, feel free to modify accordingly!
