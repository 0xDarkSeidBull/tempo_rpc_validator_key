

# 🚀 Tempo RPC Node Setup Guide (Production Ready)

Tempo is a high-performance blockchain backed by **Stripe & Paradigm**, designed specifically for **stablecoin payments**.
Built on the **Reth (Rust Ethereum) SDK**, Tempo delivers **sub-second finality**, **Ethereum-compatible RPC**, and **stablecoin-native fees**.

This guide explains how to deploy a **reliable Tempo RPC node** using **snapshots**, **Docker**, and **systemd** for production environments.

---

## ⚠️ Important Notes (Read Carefully)

* ❌ **Do NOT run an RPC node and Validator on the same server**
  Ports `30303` and `8545` overlap and will cause conflicts.
* ❌ **Do NOT reuse the same data directory** for multiple nodes
  This will corrupt the database.
* 🔐 **Validator nodes are permissioned**
  You must email **[partners@tempo.xyz](mailto:partners@tempo.xyz)** with:

  * Validator public key
  * Server public IP
* 💾 **Storage reality check**

  * Docs mention **100GB minimum**
  * Real-world usage requires **500GB+ NVMe**
  * Snapshots alone extract to **300GB+**

---

## 🧩 Tempo Node Types

| Node Type | Permission  | Purpose                    | Who Should Run     |
| --------- | ----------- | -------------------------- | ------------------ |
| RPC Node  | Open        | Chain sync, RPC APIs       | Anyone             |
| Validator | Whitelisted | Block production & rewards | Approved operators |

---

## 🖥️ System Requirements

| Component | Minimum      | Recommended      |
| --------- | ------------ | ---------------- |
| CPU       | 4 Cores      | 8+ Cores         |
| RAM       | 16 GB        | 32 GB            |
| Storage   | 500 GB NVMe  | 1 TB NVMe        |
| Bandwidth | 50 Mbps      | 100+ Mbps        |
| OS        | Ubuntu 22.04 | Ubuntu 24.04 LTS |

---

## 🔥 Required Ports

| Port  | Protocol  | Purpose            |
| ----- | --------- | ------------------ |
| 30303 | TCP + UDP | P2P networking     |
| 8545  | TCP       | HTTP RPC           |
| 8546  | TCP       | WebSocket RPC      |
| 9000  | TCP       | Metrics (optional) |

---

## ⚙️ Initial System Setup

### Update system

```bash
sudo apt update && sudo apt upgrade -y
```

### Install dependencies

```bash
sudo apt install -y \
curl git build-essential pkg-config libssl-dev \
clang lz4 jq htop
```

---

## 🚀 Increase System Limits (Required)

```bash
cat << 'EOF' | sudo tee /etc/security/limits.d/99-tempo.conf
* soft nofile 1048576
* hard nofile 1048576
EOF
```

```bash
reboot
```

---

## 📁 Create Working Directory

```bash
mkdir -p $HOME/tempo-node/data
cd $HOME/tempo-node
df -h $HOME/tempo-node
```

> Ensure **500GB+ free NVMe space**

---

## 🦀 Install Rust (Required)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

Verify:

```bash
rustc --version
cargo --version
```

---

## 🐳 Install Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker
reboot
```

Verify:

```bash
docker --version
docker run hello-world
```

---

## 🔥 Firewall Configuration (UFW)

```bash
sudo apt install -y ufw
sudo ufw allow ssh
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp
sudo ufw enable
```

---

## 📦 Install Tempo (Prebuilt Binary)

```bash
cd $HOME/tempo-node
curl -L https://tempo.xyz/install | bash
source ~/.bashrc
tempo --version
```

---

## 📸 Snapshot Sync (Highly Recommended)

> Syncing from genesis is extremely slow.

1. Visit snapshot index:

```
https://snapshots.tempoxyz.dev
```

2. Use the latest snapshot URL:

```bash
mkdir -p data
SNAP_URL="https://tempo-node-snapshots.tempoxyz.dev/tempo-XXXX.tar.lz4"
curl -L "$SNAP_URL" | lz4 -d | tar -xvf - -C $HOME/tempo-node/data
```

⚠️ Snapshot download ~200GB
⚠️ Extracted size ~300GB+

Verify:

```bash
du -sh $HOME/tempo-node/data
```

---

## 🖥️ Run Node (Test Mode)

```bash
tempo node \
  --follow \
  --http --http.port 8545 \
  --http.api eth,net,web3,txpool,trace
```

Stop with `Ctrl+C` after confirming sync & peers.

---

## ⚙️ Production Setup (systemd)

```bash
sudo tee /etc/systemd/system/tempo-rpc.service > /dev/null << EOF
[Unit]
Description=Tempo RPC Node
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME/tempo-node
Environment=RUST_LOG=info
ExecStart=/usr/local/bin/tempo node \
  --datadir $HOME/tempo-node/data \
  --port 30303 \
  --discovery.addr 0.0.0.0 \
  --discovery.port 30303 \
  --http \
  --http.addr 0.0.0.0 \
  --http.port 8545 \
  --http.api eth,net,web3,txpool,trace \
  --ws \
  --ws.addr 127.0.0.1 \
  --ws.port 8546 \
  --metrics 9000
Restart=always
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF
```

Activate:

```bash
sudo systemctl daemon-reload
sudo systemctl enable tempo-rpc
sudo systemctl start tempo-rpc
sudo systemctl status tempo-rpc
```

---

## 🧪 Validator (Application Phase)

```bash
git clone https://github.com/tempoxyz/tempo.git
cd tempo
cargo build --release --bin tempo
```

Generate keys:

```bash
./target/release/tempo consensus generate-private-key \
  --output $HOME/tempo-node/validator-key
```

Public key:

```bash
./target/release/tempo consensus calculate-public-key \
  --private-key $HOME/tempo-node/validator-key
```

📧 Send **public key + server IP** to
`partners@tempo.xyz`

---

## ✅ Quick Checklist

### RPC Node

* [x] Tempo binary installed
* [x] Snapshot synced
* [x] systemd enabled
* [x] Peers connected

### Validator (Extra)

* [ ] Keys generated
* [ ] Keys backed up
* [ ] Whitelist approved
* [ ] Reward address set
* [ ] Validator service created

---

## 📚 References

* Official Docs: [https://docs.tempo.xyz/guide/node](https://docs.tempo.xyz/guide/node)
* Snapshots: [https://snapshots.tempoxyz.dev](https://snapshots.tempoxyz.dev)
* Twitter: [https://twitter.com/web3sachin](https://twitter.com/web3sachin)

---

### 🧠 Guide Author

**web3sachin/0xDarkSeidBull**

January 2026

Tempo v0.8.0

---


