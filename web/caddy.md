# caddy
[caddy](https://caddyserver.com/) is a webserver written in Go that can manage LetsEncrypt TLS certs automatically.

## Install
1. Grab an asset link off [Github Releases](https://github.com/caddyserver/caddy/releases)
1. Untar with `tar -xf <file>`
1. Create a caddy group
    ```bash
    sudo groupadd --system caddy
    ```
1. Create a caddy user
    ```bash
    sudo useradd --system \
    	--gid caddy \
    	--create-home \
    	--home-dir /home/caddy \
    	--shell /usr/sbin/nologin \
    	--comment "Caddy web server" \
    	caddy
    ```
1. Write a systemd service file
    ```
    [Unit]
    Description=Caddy
    Documentation=https://caddyserver.com/docs/
    After=network.target

    [Service]
    User=caddy
    Group=caddy
    ExecStart=/home/caddy/caddy run --environ --config /home/caddy/Caddyfile
    ExecReload=/home/caddy/caddy reload --config /home/caddy/Caddyfile
    TimeoutStopSec=5s
    LimitNOFILE=1048576
    LimitNPROC=512
    PrivateTmp=true
    ProtectSystem=full
    AmbientCapabilities=CAP_NET_BIND_SERVICE

    [Install]
    WantedBy=multi-user.target
    ```
1. Write a config file
    ```bash
    $ cat Caddyfile
    ns1.choo.dev
    reverse_proxy 127.0.0.1:8080
    ```
1. Move files to caddy's home and change permissions
    ```bash
    sudo mv caddy Caddyfile /home/caddy/
    sudo chown -R caddy:caddy /home/caddy
    ```
1. Move the systemd file to the write location
    ```bash
    sudo mv caddy.service /etc/systemd/system/caddy.service
    ```
1. Run the thing
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable caddy
    sudo systemctl start caddy
    ```
