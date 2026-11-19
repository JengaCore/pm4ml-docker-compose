# Payment Manager - Docker Compose Stack

This repository provides a **Docker Compose stack** for running the **Payment Manager** in a **lightweight, single-machine production setup**.  
It is designed for **DFSPs (Digital Financial Service Providers)** who need a simplified deployment alternative to Kubernetes-based infrastructure.

## Features
- PM4ML Core Services
- Keycloak
- API Gateway
- Vault
- Other core dependencies bundled together

## Usage

### 1. DNS Configuration

   You need to set up DNS records for your domain to point to the server where you will run this stack. The following subdomains should be configured:
   - `portal.<YOUR_DOMAIN>` - for accessing the Payment Manager Portal
   - `ml-connect.<YOUR_DOMAIN>` - for Switch to access the ML Connect service
   - `ttk.<YOUR_DOMAIN>` - for accessing the TTK

   It is recommended to use wildcard DNS records for easier management.

### 2. Firewall Configuration

   Allow incoming traffic on the following ports:
   
   **Required Ports:**
   - `443` (HTTPS) - for secure access to the Payment Manager Portal and the same port is used for connecting to ML Switch

   **Optional Ports:**
   - `80` (HTTP) - for certificate generation (can be disabled after setup)
   - `5050` & `6060` (TTK) - for accessing the TTK (can be restricted to user IPs)


### 3. Clone the Repository

   ```bash
   git clone https://github.com/pm4ml/pm4ml-docker-compose.git
   cd pm4ml-docker-compose
   sudo usermod -aG docker $USER
   newgrp docker
   ```

### 4. Populate .env file

   Create a copy of the `.env.example` file and rename it to `.env`. Update the environment variables in the `.env` file as needed.

### 5. SSL Certificate Setup (Required)
    
    Before starting the services, generate SSL certificates for HAProxy

    Note: For generating certificates, ensure that port 80 is open and accessible from the internet.
    It can be disabled after certificate generation.
    
    Stop HAProxy Container if running
    ```bash
    docker compose stop haproxy
    ```
    Run Certificate Generation
    ```bash
    docker compose -f docker-compose-certbot.yaml up
    ```
    This will obtain certificates for your domain and make it available for HAProxy

### 6. Start Vault

   ```bash
   docker compose --profile vault up -d
   ```

### 7. Backup Vault Unseal keys and Root Token

   > **_Note: Vault needs to be unsealed everytime it is started. You need to provide the unseal keys generated during the first initialization. So make sure to store them (unseal keys and root token) securely._**

   - Wait for the `init-vault` container to complete initialization before proceeding.

      ```bash
      docker ps -a
      docker logs init-vault
      ```
   - Find the unseal keys and root token in the logs of the `init-vault` container and store them securely.

### 8. Create Vault Secrets

   After starting the services, you need to create the necessary Vault secrets for the Payment Manager to function correctly. Use the following command to create secrets one at a time:

   ```bash
   docker compose exec vault-agent /vault/create-secrets.sh <SECRET_NAME>
   ```

   Required secrets:
   ```bash
   docker compose exec vault-agent /vault/create-secrets.sh AUTH_CLIENT_SECRET
   docker compose exec vault-agent /vault/create-secrets.sh OAUTH_CLIENT_SECRET
   docker compose exec vault-agent /vault/create-secrets.sh PORTAL_PASSWORD
   ```

   Additional secrets if needed based on the core-connector configuration:
   ```bash
   docker compose exec vault-agent /vault/create-secrets.sh CC_TOKEN
   ```

   Optional, recommended: where the certificate check posts its alerts, a
   Slack-compatible incoming webhook URL (Slack, Mattermost, Rocket.Chat, a Teams
   workflow). See [Certificate renewal](#certificate-renewal).
   ```bash
   docker compose exec vault-agent /vault/create-secrets.sh ALERT_WEBHOOK_URL
   ```

   The script will prompt you to enter the secret value. All secrets are stored at `shared-secrets/pm4ml` in Vault.

### 9. Start Services

   _Note: Wait for Vault to initialize and render the secrets before starting other services._

   ```bash
   docker compose --profile pm4ml up -d
   ```

   **Additional Profiles:**
   - `--profile core-connector` (for core-connector service)
   - `--profile admin` (for portainer service for debugging purposes)
   - `--profile ttk` (for testing toolkit for testing purposes)


## Certificate renewal

The payment manager's client certificate, the one the switch checks on every
payment you send, is signed by the hub in MCM. It expires (90 days on the
sandbox). management-api asks MCM for a new one **7 days before** expiry, and
then waits: **the hub operator has to sign it**, as for the first one, and MCM
notifies nobody that a request is waiting. Unsigned at expiry, you can no longer
send payments; you still receive them.

The `cert-check` service (profile `pm4ml`) watches this for you. Every 6 hours it
reads, with your MCM credentials, what MCM holds for your DFSP, and logs one line:

| Event | Meaning | What to do |
|---|---|---|
| `cert.ok` | Signed, more than `CERT_WARN_DAYS` (14) left | Nothing |
| `cert.expiring` | Fewer than 14 days left | The renewal request comes at 7 days; warn the hub it is coming |
| `renewal.pending` | A renewal request waits for the hub's signature | **Tell the hub operator** through your agreed support channel, and ask them to sign it |
| `cert.expired` | Expired: outgoing payments fail | Escalate to the hub operator now |

On anything but `cert.ok` the container turns **unhealthy** (`docker compose ps`,
Portainer) and, if `ALERT_WEBHOOK_URL` is set in Vault (step 8), it posts the message
there. If the check itself cannot run (`check.failed`: MCM or Keycloak unreachable, the
client secret rejected), it says so once on the webhook, on the first failed run after
a good one, and the container stays unhealthy until a run succeeds. Changing
`AUTH_CLIENT_SECRET` or `ALERT_WEBHOOK_URL` in Vault needs no restart: the check reads
them again before every run. Once the hub has signed, management-api installs the new certificate within
a minute, without a restart.

```bash
docker compose logs cert-check           # the last result
docker compose ps cert-check             # healthy / unhealthy
```

### Renewing or replacing the certificate by hand

Renewal is automatic on your side; you can still start one yourself at any time
(after a suspected key leak, before a planned change, or to rehearse the procedure):

```bash
docker compose exec management-api wget -qO- --header 'Content-Type: application/json' \
  --post-data '{"reason":"<why>"}' http://127.0.0.1:9000/recreate/outboundTLS
```

The payment manager creates a new key and signing request and sends it to MCM;
`cert-check` reports `renewal.pending`; ask the hub operator to sign it; the new
certificate is installed within about two minutes, without a restart. The old one is
not revoked and stays valid until it expires: tell the hub if the reason is a leaked
key. `/recreate/JWS` does the same for the JWS signing key. The callback (server)
certificate renews itself 2 days before it expires.

Settings in `.env`: `CERT_WARN_DAYS` (14), `CERT_CHECK_INTERVAL_SECONDS` (21600).
The hub watches the same thing on its side; the alert on yours is so that the
person who can chase it, you, hears about it too.

## Accessing Services

- **Payment Manager Portal**: `https://portal.<YOUR_DOMAIN>`
- **TTK UI**: `http://ttk.<YOUR_DOMAIN>:6060`

## Unsealing Vault

To unseal Vault, you need to access the Vault UI on port `8200` or use the Vault CLI. Use THREE of the unseal keys generated during the initialization process.

You need to run the unseal command three times with different unseal keys:

```bash
docker exec -it vault vault operator unseal
```
After properly unsealead you can confirm by running the following command
```
curl -s http://127.0.0.1:8233/v1/sys/health
```

## Auto-unseal Vault
Refer to [VAULT-AUTO-UNSEAL.md](./VAULT-AUTO-UNSEAL.md) for instructions on setting up auto-unseal for Vault using various methods.

## Trouble Shooting

### A. Error when using proxmox container templates
If you are using proxmox container templates and getting the following error
```
Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: open sysctl net.ipv4.ip_unprivileged_port_start file: reopen fd 8: permission denied
```

Set the following paramters in your LXC configuration of the container tempalte in proxmox host and restart the CT.
```
unprivileged: 0
lxc.apparmor.profile: unconfined
```
### B. Get latest vault unseal key
If you for unfortunate reason restart vault unexpectedly without noticing you can get the latest unseal key by running the following command
```
docker logs init-vault 2>&1 | grep -E '^(Unseal Key [1-5]:|Initial Root Token:)'
```