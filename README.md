# ephemeral-root-ca

An on-demand root certificate authority using OpenBao and self-sealed with local age encryption. This brings up the vault instance via docker, unseals and configures the root ca with local files that are then age encrypted before being commit into git. The only thing required to bring things back online to rotate/sign certs is the original age private key (`age.key`) and a container engine.

## Overview

This project provides a portable, self-contained Root CA that can be:
- Started on-demand when you need to sign intermediate CAs
- Sealed and encrypted when not in use
- Stored safely in git (encrypted secrets only)
- Used offline after initial setup

## Prerequisites

### Option 1: Using mise (recommended)
```bash
# Install mise: https://mise.jdx.dev/getting-started.html
curl https://mise.run | sh

# While in project folder
mise trust

# Install tools
mise install -y
```

### Option 2: Manual installation

- [Task](https://taskfile.dev/) - Task runner
- [age](https://github.com/FiloSottile/age) - Modern encryption tool
- [jq](https://jqlang.github.io/jq/) - JSON processor
- [Docker](https://docs.docker.com/get-docker/) with Docker Compose
- [OpenSSL](https://www.openssl.org/) - For certificate operations (usually pre-installed)
- [curl](https://curl.se/) - For API calls (usually pre-installed)

## Quick Start

### 1. Initialize (first time only)

If you are creating your new (mostly) offline root CA you will need to bring Vault online and create a self-signed certificate. First personalize the root cert by editing the `./config/root-ca.json` (and optionally the `./config/sub-ca.json` if you are playing with creating that for testing) to your liking. Then create your AGE key.

```bash
# Generate age encryption key
task init
```
> ⚠️ **IMPORTANT**: Back up `age.key` securely! It's required to decrypt your secrets and is never committed to git.

### 2. Start OpenBao and Setup Root CA
```bash
# Start OpenBao container
docker compose up -d

task initialize

# Wait for OpenBao to be ready, then set up the Root CA
task setup-root-ca

> The setup task generates a local `secrets/root-ca.key` and matching `secrets/root-ca.pem` bundle before uploading them into OpenBao. Always seal/encrypt (`task stop`) before committing changes so the private key stays protected.
```

### 3. Sign an Intermediate CA (when needed)
```bash
# Start if not running
task start

# Sign an intermediate CA CSR
task sign-intermediate CSR_FILE=path/to/intermediate.csr
```

### 4. Stop and Seal
```bash
# Seal OpenBao and encrypt all secrets (ALWAYS do this before git commit!)
task stop
```

## Available Tasks

Enter `task` to see all tasks available.

## Directory Structure

```
.
├── Taskfile.yml          # Task definitions
├── docker-compose.yml    # OpenBao container config
├── .mise.toml            # Tool versions
├── config/
│   ├── root-ca.json      # Root CA certificate configuration
│   ├── sub-ca.json       # Sub CA certificate configuration (optional)
│   └── openbao.hcl       # OpenBao server config
├── encrypted/            # Age-encrypted secrets (safe to commit)
│   ├── init.json.age     # Encrypted unseal keys and root token
│   ├── root-ca.pem.age   # Encrypted root CA certificate
│   ├── root-ca.key.age   # Encrypted root CA private key
│   └── crl.pem.age       # Encrypted CRL
├── secrets/              # Decrypted secrets (gitignored)
│   ├── init.json         # Unseal keys and root token
│   ├── root-ca.pem       # Root CA certificate
│   ├── root-ca.key       # Root CA private key (encrypt before committing!)
│   ├── sub-ca.pem        # Subordinate CA certificate chain
│   ├── sub-ca.key        # Subordinate CA private key (encrypt before committing!)
│   └── crl.pem           # Certificate Revocation List
├── data/                 # OpenBao persistent data (gitignored)
└── age.key               # Age private key (gitignored, BACK THIS UP!)
```

## Workflow

### Initial Setup (Online)
1. Run `task init` to create age encryption key
2. Run `docker compose up -d` to start OpenBao
3. Run `task setup-root-ca` to initialize and create root CA
4. Run `task stop` to seal and encrypt everything
5. Commit encrypted files to git
6. Backup `age.key` to a secure location

### Sign Intermediate CA (Can be offline after initial setup)
1. Run `task start` to decrypt and start OpenBao
2. Run `task sign-intermediate CSR_FILE=your-csr.csr`
3. Run `task stop` to seal and encrypt

### Offline Usage
After the initial setup, you can:
1. Clone the repo on an air-gapped machine
2. Restore your `age.key`
3. Run `task start` and `task stop` without network access

## Security Considerations

- **Never commit `age.key`** - This file is your encryption key
- **Backup `age.key` securely** - Without it, you cannot decrypt your secrets
- **Keep the Root CA offline** - Only bring it online when needed
- **Treat `secrets/root-ca.key` like crown jewels** - Always run `task stop` (or `task encrypt-secrets`) before pushing so the private key remains encrypted at rest in version control
- **Use strong passphrases** - Consider age's passphrase feature for extra security
- **Rotate keys periodically** - Generate new age keys and re-encrypt

## Configuration

### Root CA Settings
Edit `config/root-ca.json` to declare the Root CA metadata:
```json
{
  "name": "Ephemeral Root CA",
  "organization": "Ephemeral PKI",
  "domain": "pki.example.com",
  "crl_base_url": "https://crl.example.com"
}
```

- `name` sets the Root CA common name used during generation.
- `organization` is optional and defaults to `Ephemeral PKI` when omitted.
- `domain` is optional; when provided it is used to build default issuing and CRL URLs. Supply either a hostname (scheme defaults to `https://`) or a full URL. When left empty the local `BAO_ADDR` value is used instead.
- `crl_base_url` is optional; use it when you want CRLs served from a different host/base path than the issuing certificate endpoint.
- When `task setup-root-ca` runs, it first creates a local private key (`secrets/root-ca.key`) and self-signed certificate bundle (`secrets/root-ca.pem`), then uploads both to OpenBao. Keep the key encrypted with `task stop`/`task encrypt-secrets` before committing.

### Subordinate CA Settings
Edit `config/sub-ca.json` to describe the issuing CA that will be signed by the root:
```json
{
  "name": "Ephemeral Issuing CA",
  "organization": "Ephemeral PKI Issuing",
  "domain": "int-ca.example.com",
  "ttl": "43800h",
  "mount": "pki_int"
}
```

- `name` is the subordinate CA common name encoded into the certificate.
- `organization` is optional; leave blank or remove to skip setting it.
- `domain` works like the root CA domain and drives the issuing/CRL URLs for the subordinate mount.
- `ttl` controls both the generated key lifetime and the signed certificate TTL (defaults to `43800h` ≈ 5 years).
- `mount` specifies the OpenBao mount where the subordinate PKI engine lives (defaults to `pki_int`).

Once configured, run `task generate-sub-ca` (with OpenBao unsealed) to provision or refresh the issuing CA. The signed certificate is stored at `secrets/sub-ca.pem` and will be encrypted alongside the other secrets when you run `task stop` or `task encrypt-secrets`.
The subordinate private key is written to `secrets/sub-ca.key` and encrypted to `encrypted/sub-ca.key.age` during `task stop`.

### OpenBao Settings
Edit `config/openbao.hcl` for OpenBao server configuration.

## Troubleshooting

### Vault UI

Go to http://localhost:8200 and enter in the token found in `secrets/init.json`.

> **Tip** Copy/Assign/Vew the root token `jq -r '.root_token' "./secrets/init.json"`

### OpenBao won't start
```bash
# Check container logs
docker compose logs openbao

# Check if port 8200 is in use
lsof -i :8200
```

### Cannot unseal
```bash
# Ensure secrets are decrypted
task decrypt-secrets

# Check if init.json exists
cat secrets/init.json | jq .
```

### Age decryption fails
```bash
# Verify age.key exists and matches
age-keygen -y age.key  # Shows public key
```

## License

MIT
