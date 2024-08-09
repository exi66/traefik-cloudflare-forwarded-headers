# Get Real Visitor IP Address (Restoring Visitor IPs) with Traefik and CloudFlare
This project aims to modify your traefik configuration to let you get the real ip address of your visitors for your web applications that behind of Cloudflare's reverse proxy network. Bash script can be scheduled to create an automated up-to-date Cloudflare ip list file.

To provide the client (visitor) IP address for every request to the origin, Cloudflare adds the "CF-Connecting-IP" header. We will catch the header and get the real ip address of the visitor.

## Traefik Configuration
With a small configuration modification we can integrate replacing the real ip address of the visitor instead of getting CloudFlare's load balancers' ip addresses.

The bash script may run manually or can be scheduled to refresh the ip list of CloudFlare automatically.

This example uses a Traefik instance at [Dokploy](https://github.com/Dokploy/dokploy)
```sh
#!/bin/bash

# a package yq (https://github.com/mikefarah/yq) is required for operation
# you can install it with the command:
#
#       wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/bin/yq && chmod +x /usr/bin/yq
#

# for example, here is the path to traefik in the dokploy (https://github.com/Dokploy/dokploy) instance
CONFIG_FILE_PATH=${1:-/etc/dokploy/traefik/traefik.yml}
BACKUP_FILE_PATH=${2:-/etc/dokploy/traefik/traefik.yml.bak}
DOKPLOY_API_TOKEN=${3}

cat $CONFIG_FILE_PATH > $BACKUP_FILE_PATH;

yq -i "del(.entryPoints.websecure.forwardedHeaders.trustedIPs)" $CONFIG_FILE_PATH;

for i in `curl -s -L https://www.cloudflare.com/ips-v4`; do
    yq -i ".entryPoints.websecure.forwardedHeaders.trustedIPs += [\"$i\"]" $CONFIG_FILE_PATH;
done

for i in `curl -s -L https://www.cloudflare.com/ips-v6`; do
    yq -i ".entryPoints.websecure.forwardedHeaders.trustedIPs += [\"$i\"]" $CONFIG_FILE_PATH;
done

# reload traefik instance
# for example, here is the API endpoint for reloading traefik in dokploy (https://github.com/Dokploy/dokploy)
curl -X 'POST' \
  "http://localhost:3000/api/settings.reloadTraefik" \
  -H "accept: application/json" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN"
```

## Output
Your "/etc/dokploy/traefik/traefik.yml" file may look like as below;

``` yml
providers:
  docker:
    exposedByDefault: false
  file:
    directory: /etc/dokploy/traefik/dynamic
    watch: true
entryPoints:
  web:
    address: ':80'
  websecure:
    address: ':443'
    http:
      tls:
        certResolver: letsencrypt
    forwardedHeaders:
      trustedIPs:
        - 173.245.48.0/20
        - 103.21.244.0/22
        - 103.22.200.0/22
        - 103.31.4.0/22
        - 141.101.64.0/18
        - 108.162.192.0/18
        - 190.93.240.0/20
        - 188.114.96.0/20
        - 197.234.240.0/22
        - 198.41.128.0/17
        - 162.158.0.0/15
        - 104.16.0.0/13
        - 104.24.0.0/14
        - 172.64.0.0/13
        - 131.0.72.0/22
        - 2400:cb00::/32
        - 2606:4700::/32
        - 2803:f800::/32
        - 2405:b500::/32
        - 2405:8100::/32
        - 2a06:98c0::/29
        - 2c0f:f248::/32
api:
  insecure: true
certificatesResolvers:
  letsencrypt:
    acme:
      email: root@example.com
      storage: /etc/dokploy/traefik/dynamic/acme.json
      httpChallenge:
        entryPoint: web
```

## Crontab
Change the location of "/opt/scripts/cloudflare-ip-whitelist-sync.sh" anywhere you want. 
CloudFlare ip addresses are automatically refreshed every day, and traefik will be realoded when synchronization is completed.
```sh
# Auto sync ip addresses of Cloudflare and reload traefik
30 2 * * * /opt/scripts/cloudflare-ip-whitelist-sync.sh >/dev/null 2>&1
```

### License

[Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)


### DISCLAIMER
----------
Please note: all tools/ scripts in this repo are released for use "AS IS" **without any warranties of any kind**,
including, but not limited to their installation, use, or performance.  We disclaim any and all warranties, either 
express or implied, including but not limited to any warranty of noninfringement, merchantability, and/ or fitness 
for a particular purpose.  We do not warrant that the technology will meet your requirements, that the operation 
thereof will be uninterrupted or error-free, or that any errors will be corrected.

Any use of these scripts and tools is **at your own risk**.  There is no guarantee that they have been through 
thorough testing in a comparable environment and we are not responsible for any damage or data loss incurred with 
their use.

You are responsible for reviewing and testing any scripts you run *thoroughly* before use in any non-testing 
environment.
