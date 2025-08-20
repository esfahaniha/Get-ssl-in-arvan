# Automated Let’s Encrypt SSL for `test.example.com` via ArvanCloud DNS

This guide explains how to issue and automatically renew SSL certificates for `test.example.com` using Let’s Encrypt, Certbot, and ArvanCloud DNS. It includes automation for DNS TXT record creation and removal.

---

## Requirements

Make sure the following are installed:

1. **Python 3** (required for Certbot)

   ```bash
   sudo apt update
   sudo apt install python3 python3-venv python3-pip -y
   ```

2. **Certbot** (ACME client)

   ```bash
   sudo apt install certbot -y
   ```

3. **curl** (for API interaction)

   ```bash
   sudo apt install curl -y
   ```

4. **jq** (optional but recommended for parsing JSON)

   ```bash
   sudo apt install jq -y
   ```

5. **bash** (for scripting, usually pre-installed)

---

## Step 1: Create ArvanCloud API Credentials

1. Login to [ArvanCloud Panel](https://panel.arvancloud.ir/).
2. Go to **API Keys** → create a new key.
3. Save the API key securely. It will be used in scripts.

Create a credentials file `/etc/letsencrypt/arvancloud.ini`:

```ini
# /etc/letsencrypt/arvancloud.ini
arvancloud_api_key = "YOUR_API_KEY_HERE"
```

Set secure permissions:

```bash
sudo chmod 600 /etc/letsencrypt/arvancloud.ini
```

---

## Step 2: Create Automation Scripts

### 2.1 Authentication Script (`auth-arvancloud.sh`)

```bash
#!/bin/bash
# /etc/letsencrypt/auth-arvancloud.sh
set -e

DOMAIN="$CERTBOT_DOMAIN"
TXT_VALUE="$CERTBOT_VALIDATION"
API_KEY="$(grep arvancloud_api_key /etc/letsencrypt/arvancloud.ini | cut -d'=' -f2 | tr -d ' ' )"

curl --location 'https://napi.arvancloud.ir/cdn/4.0/domains/serversamin.ir/dns-records' \
--header "Authorization: $API_KEY" \
--header 'Content-Type: application/json' \
--data '{"type":"TXT","name":"_acme-challenge.'$DOMAIN'","cloud":false,"value":{"text":"'$TXT_VALUE'"},"ttl":120}'
```

Make it executable:

```bash
sudo chmod +x /etc/letsencrypt/auth-arvancloud.sh
```

### 2.2 Cleanup Script (`clean-arvancloud.sh`)

```bash
#!/bin/bash
# /etc/letsencrypt/clean-arvancloud.sh
set -e

DOMAIN="$CERTBOT_DOMAIN"
API_KEY="$(grep arvancloud_api_key /etc/letsencrypt/arvancloud.ini | cut -d'=' -f2 | tr -d ' ' )"

# Fetch all TXT records
RECORDS=$(curl --silent --location 'https://napi.arvancloud.ir/cdn/4.0/domains/serversamin.ir/dns-records' \
--header "Authorization: $API_KEY" | jq -r '.[] | select(.type=="TXT" and .name=="_acme-challenge.'$DOMAIN'") | .id')

# Delete each TXT record
for ID in $RECORDS; do
  curl --silent --location -X DELETE "https://napi.arvancloud.ir/cdn/4.0/domains/serversamin.ir/dns-records/$ID" \
  --header "Authorization: $API_KEY"
done
```

Make it executable:

```bash
sudo chmod +x /etc/letsencrypt/clean-arvancloud.sh
```

---

## Step 3: Issue Certificate

Run Certbot with manual DNS automation:

```bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --manual-auth-hook /etc/letsencrypt/auth-arvancloud.sh \
  --manual-cleanup-hook /etc/letsencrypt/clean-arvancloud.sh \
  -d test.example.com \
  --agree-tos \
  --email test@samin.com \
  --non-interactive
```

To force renewal even if the certificate is valid:

```bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --manual-auth-hook /etc/letsencrypt/auth-arvancloud.sh \
  --manual-cleanup-hook /etc/letsencrypt/clean-arvancloud.sh \
  -d test.example.com \
  --agree-tos \
  --email test@samin.com \
  --non-interactive \
  --force-renewal
```

---

## Step 4: Automate Renewal (Optional)

Add a cron job to run renewals automatically (Certbot will use the hooks for DNS update):

```bash
sudo crontab -e
```

Add:

```cron
0 3 * * * certbot renew --quiet
```

This checks and renews certificates daily at 3 AM.

---

## Step 5: Verify Certificates

Check issued certificates:

```bash
sudo certbot certificates
```

Files are located under:

* Certificate: `/etc/letsencrypt/live/test.example.com/fullchain.pem`
* Private Key: `/etc/letsencrypt/live/test.example.com/privkey.pem`

You can now use them with your ELK stack or other services.

---

**Notes:**

* The `_acme-challenge` TXT record is automatically created and removed during issuance.
* Ensure that your server has internet access to reach `napi.arvancloud.ir`.
* Keep your API key secure; anyone with it can modify your DNS records.
