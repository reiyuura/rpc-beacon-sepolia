# Complete Guide: Running an Ethereum Sepolia RPC Node with Geth and Lighthouse on a VPS

This guide aims to walk you through the process of setting up a full Ethereum Sepolia RPC node using Geth as the execution client and Lighthouse as the consensus client on a Linux-based Virtual Private Server (VPS).

This process requires attention to detail and patience, especially during the synchronization phases.

## Table of Contents
1.  [System Requirements](#system-requirements)
2.  [Step 1: Initial VPS Setup](#step-1-initial-vps-setup)
3.  [Step 2: Installing and Configuring Geth (Execution Client)](#step-2-installing-and-configuring-geth-execution-client)
4.  [Step 3: Installing and Configuring Lighthouse (Consensus Client)](#step-3-installing-and-configuring-lighthouse-consensus-client)
5.  [Step 4: Resolving JWT Permissions](#step-4-resolving-jwt-permissions)
6.  [Step 5: Troubleshooting DNS and SSL Issues (Lessons Learned)](#step-5-troubleshooting-dns-and-ssl-issues-lessons-learned)
7.  [Step 6: Starting and Monitoring the Nodes](#step-6-starting-and-monitoring-the-nodes)
8.  [Step 7: Accessing the RPC Node](#step-7-accessing-the-rpc-node)
9.  [Step 8: Updating Clients (Lighthouse Example)](#step-8-updating-clients-lighthouse-example)
10. [Additional Tips and General Troubleshooting](#additional-tips-and-general-troubleshooting)

## System Requirements

* **VPS (Virtual Private Server):**
    * CPU: Minimum 2 cores (4 cores or more recommended)
    * RAM: Minimum 8 GB (16 GB or more recommended)
    * Storage: Minimum **250 GB SSD** (NVMe SSD highly recommended for better I/O performance)
    * Bandwidth: Stable internet connection with a large data cap or unmetered.
* **Operating System:** Ubuntu 22.04 LTS (or other modern Linux distros, though commands might slightly differ).
* **Access:** Root access or a user with `sudo` privileges.

## Step 1: Initial VPS Setup

1.  **Login to your VPS via SSH:**
    ```bash
    ssh username@YOUR_VPS_IP_ADDRESS
    ```

2.  **Update System:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

3.  **Install Basic Required Packages:**
    ```bash
    sudo apt install -y curl git build-essential libssl-dev pkg-config screen libclang-dev cmake ufw
    ```

4.  **Install Go (Golang):**
    Visit the [official Go downloads page](https://golang.org/dl/) to get the latest version link. Replace `GO_VERSION` below if necessary.
    ```bash
    GO_VERSION="1.22.3" # Adjust to the latest stable version
    wget [https://golang.org/dl/go$](https://golang.org/dl/go$){GO_VERSION}.linux-amd64.tar.gz
    sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
    echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.profile
    echo 'export GOPATH=$HOME/go' >> ~/.profile # Optional, if you plan to develop with Go
    echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.profile # Optional
    source ~/.profile
    go version # Verify installation
    ```

5.  **Install Rust:**
    ```bash
    curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs](https://sh.rustup.rs) | sh
    ```
    Follow the on-screen instructions (usually select option 1 for default installation). Then, configure the path:
    ```bash
    source $HOME/.cargo/env
    rustc --version # Verify installation
    ```

6.  **Configure Firewall (UFW - Uncomplicated Firewall):**
    Open the necessary ports:
    * Geth P2P: `30303` (TCP & UDP)
    * Geth HTTP RPC: `8545` (TCP) - *Be cautious if exposing publicly!*
    * Geth WebSocket RPC: `8546` (TCP) - *Be cautious if exposing publicly!*
    * Geth Auth RPC: `8551` (TCP) - Only for local communication with the Consensus Client.
    * Lighthouse P2P: `9000` (TCP & UDP)

    ```bash
    sudo ufw allow 30303/tcp
    sudo ufw allow 30303/udp
    sudo ufw allow 8545/tcp  # Consider restricting to specific IPs if not for public use
    sudo ufw allow 8546/tcp  # Consider restricting to specific IPs if not for public use
    sudo ufw allow 8551/tcp  # Should only be accessed from localhost
    sudo ufw allow 9000/tcp
    sudo ufw allow 9000/udp
    sudo ufw allow ssh      # Ensure SSH remains allowed!
    sudo ufw enable
    sudo ufw status
    ```
    **RPC Security:** If you plan to expose Geth RPC (ports 8545/8546) to the internet, ensure you understand the security risks and consider using authentication or a stricter firewall. For local communication between Geth and Lighthouse, port `8551` does not need to be publicly open.

## Step 2: Installing and Configuring Geth (Execution Client)

1.  **Create a Dedicated User for Geth (Recommended):**
    ```bash
    sudo adduser --system --no-create-home --group gethuser
    ```

2.  **Download Geth (Pre-compiled Binary v1.15.11):**
    The version used in this example is `1.15.11-36b2371c`.
    ```bash
    cd ~ # Change to your home directory if not already there
    GETH_FILENAME="geth-linux-amd64-1.15.11-36b2371c.tar.gz"
    GETH_EXTRACTED_DIR="geth-linux-amd64-1.15.11-36b2371c"
    wget [https://gethstore.blob.core.windows.net/builds/$](https://gethstore.blob.core.windows.net/builds/$){GETH_FILENAME}
    tar -xvf ${GETH_FILENAME}
    sudo mv ${GETH_EXTRACTED_DIR}/geth /usr/local/bin/
    geth version # Verify, should display 1.15.11
    rm -rf ${GETH_FILENAME} ${GETH_EXTRACTED_DIR} # Clean up downloaded file and extracted folder
    ```

3.  **Create Geth Data Directory:**
    ```bash
    sudo mkdir -p /var/lib/geth-sepolia
    sudo chown -R gethuser:gethuser /var/lib/geth-sepolia
    ```

4.  **Create JWT Secret for Engine API:**
    This is crucial for secure communication between Geth and Lighthouse.
    ```bash
    sudo mkdir -p /var/lib/jwtsecret
    openssl rand -hex 32 | sudo tee /var/lib/jwtsecret/jwt.hex > /dev/null
    # Set initial ownership to gethuser, permissions will be adjusted later for Lighthouse access
    sudo chown -R gethuser:gethuser /var/lib/jwtsecret 
    sudo chmod 600 /var/lib/jwtsecret/jwt.hex # Initial permission, will be changed later
    ```

5.  **Create Systemd Service File for Geth (`geth-sepolia.service`):**
    Create the file `sudo nano /etc/systemd/system/geth-sepolia.service` and add the following content:
    ```ini
    [Unit]
    Description=Geth Sepolia Execution Client
    After=network.target
    Wants=network.target

    [Service]
    User=gethuser
    Group=gethuser
    Type=simple
    Restart=always
    RestartSec=5
    ExecStart=/usr/local/bin/geth \
        --sepolia \
        --datadir /var/lib/geth-sepolia \
        --http \
        --http.addr "0.0.0.0" \
        --http.port 8545 \
        --http.api "eth,net,web3,engine,admin,txpool" \
        --http.vhosts "*" \
        --ws \
        --ws.addr "0.0.0.0" \
        --ws.port 8546 \
        --ws.api "eth,net,web3,engine" \
        --ws.origins "*" \
        --authrpc.jwtsecret /var/lib/jwtsecret/jwt.hex \
        --authrpc.addr "127.0.0.1" \
        --authrpc.port 8551 \
        --authrpc.vhosts "localhost" \
        --metrics \
        --metrics.expensive \
        --cache 4096 \
        --maxpeers 50 \
        --syncmode "snap"

    StandardOutput=journal
    StandardError=journal

    [Install]
    WantedBy=multi-user.target
    ```
    * Adjust `--cache` based on your VPS RAM (e.g., `4096` for 8GB RAM, `8192` for 16GB RAM).
    * `--syncmode "snap"` is the default and recommended.

6.  **Reload Daemon and Enable Geth Service:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable geth-sepolia.service
    ```
    *(Do not start Geth yet; we need Lighthouse first)*

## Step 3: Installing and Configuring Lighthouse (Consensus Client)

1.  **Create a Dedicated User for Lighthouse (Recommended):**
    ```bash
    sudo adduser --system --no-create-home --group lighthouseuser
    ```

2.  **Download Lighthouse (Pre-compiled Binary v7.0.1):**
    The version used in this example is `v7.0.1`.
    ```bash
    cd ~ # Change to your home directory if not already there
    LIGHTHOUSE_FILENAME="lighthouse-v7.0.1-x86_64-unknown-linux-gnu.tar.gz"
    wget [https://github.com/sigp/lighthouse/releases/download/v7.0.1/$](https://github.com/sigp/lighthouse/releases/download/v7.0.1/$){LIGHTHOUSE_FILENAME}
    tar -xvf ${LIGHTHOUSE_FILENAME}
    sudo mv lighthouse /usr/local/bin/ # The extracted executable is usually named 'lighthouse'
    lighthouse --version # Verify, should display v7.0.1
    rm ${LIGHTHOUSE_FILENAME} # Clean up the downloaded archive
    ```

3.  **Create Lighthouse Data Directory:**
    ```bash
    sudo mkdir -p /var/lib/lighthouse-sepolia
    sudo chown -R lighthouseuser:lighthouseuser /var/lib/lighthouse-sepolia
    ```

4.  **Create Systemd Service File for Lighthouse (`lighthouse-sepolia-beacon.service`):**
    Create the file `sudo nano /etc/systemd/system/lighthouse-sepolia-beacon.service` and add the following content:
    ```ini
    [Unit]
    Description=Lighthouse Sepolia Consensus Beacon Node
    Wants=geth-sepolia.service
    After=geth-sepolia.service

    [Service]
    User=lighthouseuser
    Group=lighthouseuser
    Type=simple
    Restart=always
    RestartSec=5
    ExecStart=/usr/local/bin/lighthouse beacon_node \
        --datadir /var/lib/lighthouse-sepolia \
        --network sepolia \
        --execution-endpoint [http://127.0.0.1:8551](http://127.0.0.1:8551) \
        --execution-jwt /var/lib/jwtsecret/jwt.hex \
        --http \
        --http-address "0.0.0.0" \
        --http-port 5052 \
        --metrics \
        --metrics-address "0.0.0.0" \
        --metrics-port 5054 \
        --checkpoint-sync-url [https://beaconstate-sepolia.chainsafe.io/](https://beaconstate-sepolia.chainsafe.io/)

    StandardOutput=journal
    StandardError=journal

    [Install]
    WantedBy=multi-user.target
    ```
    **IMPORTANT:** `--checkpoint-sync-url https://beaconstate-sepolia.chainsafe.io/` was found to work after other URLs caused SSL issues.

5.  **Reload Daemon and Enable Lighthouse Service:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable lighthouse-sepolia-beacon.service
    ```

## Step 4: Resolving JWT Permissions

Lighthouse (running as `lighthouseuser`) needs to read the `jwt.hex` file owned by `gethuser`.
Set the ownership and permissions correctly:

1.  Ensure JWT directory and file ownership is `gethuser:gethuser`:
    ```bash
    sudo chown gethuser:gethuser /var/lib/jwtsecret
    sudo chown gethuser:gethuser /var/lib/jwtsecret/jwt.hex
    ```
2.  Set directory permissions for the `gethuser` group to access it:
    ```bash
    sudo chmod 750 /var/lib/jwtsecret 
    ```
    *(Owner: rwx, Group: rx, Others: ---)*
3.  Set JWT file permissions for the `gethuser` group to read it:
    ```bash
    sudo chmod 640 /var/lib/jwtsecret/jwt.hex
    ```
    *(Owner: rw, Group: r, Others: ---)*
4.  Add `lighthouseuser` to the `gethuser` group:
    ```bash
    sudo usermod -a -G gethuser lighthouseuser
    ```
    You may need to reboot or at least restart both services for this group change to take full effect.

## Step 5: Troubleshooting DNS and SSL Issues (Lessons Learned)

If you encounter connection, DNS, or SSL problems with Lighthouse:

1.  **Configure `systemd-resolved` for Reliable DNS:**
    If `nslookup target_hostname` (which uses the system resolver at `127.0.0.53`) fails with `NXDOMAIN` or similar errors:
    a.  Edit the `systemd-resolved` configuration file:
        ```bash
        sudo nano /etc/systemd/resolved.conf
        ```
    b.  Under the `[Resolve]` section, uncomment or add the following lines:
        ```ini
        DNS=8.8.8.8 1.1.1.1
        FallbackDNS=8.8.4.4 1.0.0.1
        Domains=~.
        ```
    c.  Save the file, then restart the `systemd-resolved` service:
        ```bash
        sudo systemctl restart systemd-resolved.service
        ```
    d.  Verify DNS again with `nslookup target_hostname`. It should now succeed.

2.  **Test SSL Connection and Verification at the System Level:**
    Use `openssl s_client` to test SSL connections to HTTPS endpoints directly and verify their certificates from the OS perspective:
    ```bash
    openssl s_client -connect TARGET_HOST:443 -servername TARGET_HOST
    ```
    Example: `openssl s_client -connect sepolia.checkpoint.sigp.io:443 -servername sepolia.checkpoint.sigp.io`
    Observe the output, especially the `Verify return code:`. `0 (ok)` means the certificate was successfully verified by your system. If there's an error, it indicates an issue with CA certificates or SSL validation at the OS level.

3.  **Ensure Latest CA Certificates:**
    ```bash
    sudo apt update && sudo apt install -y ca-certificates && sudo update-ca-certificates --fresh
    ```

4.  **Ensure Accurate System Time:**
    ```bash
    timedatectl status
    ```
    Ensure `System clock synchronized: yes`. If not, fix time synchronization (e.g., with `sudo systemctl restart systemd-timesyncd.service` if available, or `ntp`, `chrony`).

5.  **Specific Checkpoint Sync URL Issues:**
    During debugging, it was found that the URL `https://sepolia.checkpoint.sigp.io/` caused a specific SSL "hostname mismatch" error in Lighthouse, even though `openssl s_client` succeeded. Switching to `https://beaconstate-sepolia.chainsafe.io/` resolved this issue for Lighthouse. Always test DNS resolution (`nslookup`) and SSL (`openssl s_client`) for new checkpoint URLs before use.

## Step 6: Starting and Monitoring the Nodes

1.  **Start Services (Order is Important):**
    a.  Start Geth first:
        ```bash
        sudo systemctl start geth-sepolia.service
        ```
    b.  Wait about 1-2 minutes for Geth to initialize its Engine API.
    c.  Start Lighthouse:
        ```bash
        sudo systemctl start lighthouse-sepolia-beacon.service
        ```

2.  **Check Service Status:**
    ```bash
    sudo systemctl status geth-sepolia.service
    sudo systemctl status lighthouse-sepolia-beacon.service
    ```
    Both should be `active (running)`.

3.  **View Logs to Monitor Synchronization:**
    * Geth Logs: `sudo journalctl -fu geth-sepolia.service`
    * Lighthouse Logs: `sudo journalctl -fu lighthouse-sepolia-beacon.service`

    **What to Expect in Logs:**
    * **Lighthouse:** Will initiate "checkpoint sync," find peers, and eventually report "Synced" with increasing slots, epochs, and peer count.
    * **Geth:** Will show "Forkchoice requested sync to new head" (indicating communication with Lighthouse), then "Syncing beacon headers," followed by "Syncing: chain download in progress" and the longest part, "Syncing: state download in progress." Note the percentage (`synced=%`) and estimated time (`eta=`).

    **Patience:** Geth's state sync can take several hours or more. Lighthouse is usually faster with checkpoint sync.

## Step 7: Accessing the RPC Node

Once both clients are fully synced:
* **Geth HTTP RPC Endpoint:** `http://YOUR_VPS_IP_ADDRESS:8545`
* **Geth WebSocket RPC Endpoint:** `ws://YOUR_VPS_IP_ADDRESS:8546`

You can test the RPC connection locally on the VPS (if `curl` is installed):
```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' [http://127.0.0.1:8545](http://127.0.0.1:8545)
