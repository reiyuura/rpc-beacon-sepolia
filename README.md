# Panduan Lengkap: Menjalankan Node RPC Ethereum Sepolia dengan Geth dan Lighthouse di VPS

Panduan ini bertujuan untuk memandu Anda melalui proses penyiapan node RPC Ethereum Sepolia penuh menggunakan Geth sebagai execution client dan Lighthouse sebagai consensus client di sebuah Virtual Private Server (VPS) berbasis Linux.

Proses ini memerlukan ketelitian dan kesabaran, terutama selama fase sinkronisasi.

## Daftar Isi
1.  [Persyaratan Sistem](#persyaratan-sistem)
2.  [Langkah 1: Persiapan Awal VPS](#langkah-1-persiapan-awal-vps)
3.  [Langkah 2: Instalasi dan Konfigurasi Geth (Execution Client)](#langkah-2-instalasi-dan-konfigurasi-geth-execution-client)
4.  [Langkah 3: Instalasi dan Konfigurasi Lighthouse (Consensus Client)](#langkah-3-instalasi-dan-konfigurasi-lighthouse-consensus-client)
5.  [Langkah 4: Mengatasi Masalah Izin JWT](#langkah-4-mengatasi-masalah-izin-jwt)
6.  [Langkah 5: Troubleshooting Masalah DNS dan SSL (Pelajaran dari Debugging)](#langkah-5-troubleshooting-masalah-dns-dan-ssl-pelajaran-dari-debugging)
7.  [Langkah 6: Memulai dan Memantau Node](#langkah-6-memulai-dan-memantau-node)
8.  [Langkah 7: Mengakses RPC Node](#langkah-7-mengakses-rpc-node)
9.  [Langkah 8: Update Klien (Contoh Lighthouse)](#langkah-8-update-klien-contoh-lighthouse)
10. [Tips Tambahan dan Troubleshooting Umum](#tips-tambahan-dan-troubleshooting-umum)

## Persyaratan Sistem

* **VPS (Virtual Private Server):**
    * CPU: Minimal 2 core (4 core atau lebih direkomendasikan)
    * RAM: Minimal 8 GB (16 GB atau lebih direkomendasikan)
    * Penyimpanan: Minimal **250 GB SSD** (NVMe SSD sangat direkomendasikan untuk performa I/O yang lebih baik)
    * Bandwidth: Koneksi internet stabil dengan kuota yang cukup besar atau unmetered.
* **Sistem Operasi:** Ubuntu 22.04 LTS (atau distro Linux modern lainnya, namun perintah mungkin sedikit berbeda).
* **Akses:** Akses root atau pengguna dengan hak `sudo`.

## Langkah 1: Persiapan Awal VPS

1.  **Login ke VPS Anda melalui SSH:**
    ```bash
    ssh username@ALAMAT_IP_VPS_ANDA
    ```

2.  **Update Sistem:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

3.  **Install Paket Dasar yang Diperlukan:**
    ```bash
    sudo apt install -y curl git build-essential libssl-dev pkg-config screen libclang-dev cmake ufw
    ```

4.  **Install Go (Golang):**
    Kunjungi [halaman unduhan Go resmi](https://golang.org/dl/) untuk mendapatkan link versi terbaru. Ganti `GO_VERSION` di bawah jika perlu.
    ```bash
    GO_VERSION="1.22.3" # Sesuaikan dengan versi stabil terbaru
    wget [https://golang.org/dl/go$](https://golang.org/dl/go$){GO_VERSION}.linux-amd64.tar.gz
    sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
    echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.profile
    echo 'export GOPATH=$HOME/go' >> ~/.profile # Opsional, jika Anda berencana mengembangkan dengan Go
    echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.profile # Opsional
    source ~/.profile
    go version # Verifikasi instalasi
    ```

5.  **Install Rust:**
    ```bash
    curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs](https://sh.rustup.rs) | sh
    ```
    Ikuti instruksi di layar (biasanya pilih opsi 1 untuk instalasi default). Kemudian, konfigurasikan path:
    ```bash
    source $HOME/.cargo/env
    rustc --version # Verifikasi instalasi
    ```

6.  **Konfigurasi Firewall (UFW - Uncomplicated Firewall):**
    Buka port yang diperlukan:
    * Geth P2P: `30303` (TCP & UDP)
    * Geth HTTP RPC: `8545` (TCP) - *Hati-hati jika diekspos ke publik!*
    * Geth WebSocket RPC: `8546` (TCP) - *Hati-hati jika diekspos ke publik!*
    * Geth Auth RPC: `8551` (TCP) - Hanya untuk komunikasi lokal dengan Consensus Client.
    * Lighthouse P2P: `9000` (TCP & UDP)

    ```bash
    sudo ufw allow 30303/tcp
    sudo ufw allow 30303/udp
    sudo ufw allow 8545/tcp  # Pertimbangkan untuk membatasi ke IP tertentu jika tidak untuk publik
    sudo ufw allow 8546/tcp  # Pertimbangkan untuk membatasi ke IP tertentu jika tidak untuk publik
    sudo ufw allow 8551/tcp  # Seharusnya hanya diakses dari localhost
    sudo ufw allow 9000/tcp
    sudo ufw allow 9000/udp
    sudo ufw allow ssh      # Pastikan SSH tetap diizinkan!
    sudo ufw enable
    sudo ufw status
    ```
    **Keamanan RPC:** Jika Anda berencana mengekspos RPC Geth (port 8545/8546) ke internet, pastikan Anda memahami risiko keamanannya dan pertimbangkan untuk menggunakan otentikasi atau firewall yang lebih ketat. Untuk komunikasi lokal antara Geth dan Lighthouse, port `8551` tidak perlu dibuka ke publik.

## Langkah 2: Instalasi dan Konfigurasi Geth (Execution Client)

1.  **Buat Pengguna Khusus untuk Geth (Direkomendasikan):**
    ```bash
    sudo adduser --system --no-create-home --group gethuser
    ```

2.  **Download Geth (Binary Pre-compiled v1.15.11):**
    Versi yang digunakan dalam contoh ini adalah `1.15.11-36b2371c`.
    ```bash
    cd ~ # Pindah ke direktori home Anda jika belum
    GETH_FILENAME="geth-linux-amd64-1.15.11-36b2371c.tar.gz"
    GETH_EXTRACTED_DIR="geth-linux-amd64-1.15.11-36b2371c"
    wget [https://gethstore.blob.core.windows.net/builds/$](https://gethstore.blob.core.windows.net/builds/$){GETH_FILENAME}
    tar -xvf ${GETH_FILENAME}
    sudo mv ${GETH_EXTRACTED_DIR}/geth /usr/local/bin/
    geth version # Verifikasi, seharusnya menampilkan 1.15.11
    rm -rf ${GETH_FILENAME} ${GETH_EXTRACTED_DIR} # Hapus file dan folder sisa
    ```

3.  **Buat Direktori Data Geth:**
    ```bash
    sudo mkdir -p /var/lib/geth-sepolia
    sudo chown -R gethuser:gethuser /var/lib/geth-sepolia
    ```

4.  **Buat JWT Secret untuk Engine API:**
    Ini penting untuk komunikasi aman antara Geth dan Lighthouse.
    ```bash
    sudo mkdir -p /var/lib/jwtsecret
    openssl rand -hex 32 | sudo tee /var/lib/jwtsecret/jwt.hex > /dev/null
    # Atur kepemilikan awal ke gethuser, izin akan disesuaikan nanti agar Lighthouse juga bisa baca
    sudo chown -R gethuser:gethuser /var/lib/jwtsecret 
    sudo chmod 600 /var/lib/jwtsecret/jwt.hex # Izin awal, akan diubah nanti
    ```

5.  **Buat File Systemd Service untuk Geth (`geth-sepolia.service`):**
    Buat file `sudo nano /etc/systemd/system/geth-sepolia.service` dan isi dengan:
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
    * Sesuaikan `--cache` dengan RAM VPS Anda (misalnya, `4096` untuk 8GB RAM, `8192` untuk 16GB RAM).
    * `--syncmode "snap"` adalah default dan direkomendasikan.

6.  **Reload Daemon dan Enable Service Geth:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable geth-sepolia.service
    ```
    *(Jangan jalankan Geth dulu, kita perlu Lighthouse)*

## Langkah 3: Instalasi dan Konfigurasi Lighthouse (Consensus Client)

1.  **Buat Pengguna Khusus untuk Lighthouse (Direkomendasikan):**
    ```bash
    sudo adduser --system --no-create-home --group lighthouseuser
    ```

2.  **Download Lighthouse (Binary Pre-compiled v7.0.1):**
    Versi yang digunakan dalam contoh ini adalah `v7.0.1`.
    ```bash
    cd ~ # Pindah ke direktori home Anda jika belum
    LIGHTHOUSE_FILENAME="lighthouse-v7.0.1-x86_64-unknown-linux-gnu.tar.gz"
    wget [https://github.com/sigp/lighthouse/releases/download/v7.0.1/$](https://github.com/sigp/lighthouse/releases/download/v7.0.1/$){LIGHTHOUSE_FILENAME}
    tar -xvf ${LIGHTHOUSE_FILENAME}
    sudo mv lighthouse /usr/local/bin/ # File executable hasil ekstraksi biasanya bernama 'lighthouse'
    lighthouse --version # Verifikasi, seharusnya menampilkan v7.0.1
    rm ${LIGHTHOUSE_FILENAME} # Hapus file arsip sisa
    ```

3.  **Buat Direktori Data Lighthouse:**
    ```bash
    sudo mkdir -p /var/lib/lighthouse-sepolia
    sudo chown -R lighthouseuser:lighthouseuser /var/lib/lighthouse-sepolia
    ```

4.  **Buat File Systemd Service untuk Lighthouse (`lighthouse-sepolia-beacon.service`):**
    Buat file `sudo nano /etc/systemd/system/lighthouse-sepolia-beacon.service` dan isi dengan:
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
    **PENTING:** `--checkpoint-sync-url https://beaconstate-sepolia.chainsafe.io/` terbukti bekerja setelah URL lain mengalami masalah SSL.

5.  **Reload Daemon dan Enable Service Lighthouse:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable lighthouse-sepolia-beacon.service
    ```

## Langkah 4: Mengatasi Masalah Izin JWT

Lighthouse (berjalan sebagai `lighthouseuser`) perlu membaca file `jwt.hex` yang dimiliki oleh `gethuser`.
Atur kepemilikan dan izin dengan benar:

1.  Pastikan kepemilikan direktori dan file JWT adalah `gethuser:gethuser`:
    ```bash
    sudo chown gethuser:gethuser /var/lib/jwtsecret
    sudo chown gethuser:gethuser /var/lib/jwtsecret/jwt.hex
    ```
2.  Atur izin direktori agar grup `gethuser` bisa mengaksesnya:
    ```bash
    sudo chmod 750 /var/lib/jwtsecret 
    ```
    *(Pemilik: rwx, Grup: rx, Lainnya: ---)*
3.  Atur izin file JWT agar grup `gethuser` bisa membacanya:
    ```bash
    sudo chmod 640 /var/lib/jwtsecret/jwt.hex
    ```
    *(Pemilik: rw, Grup: r, Lainnya: ---)*
4.  Tambahkan `lighthouseuser` ke grup `gethuser`:
    ```bash
    sudo usermod -a -G gethuser lighthouseuser
    ```
    Anda mungkin perlu melakukan reboot atau setidaknya me-restart kedua layanan agar perubahan grup ini efektif.

## Langkah 5: Troubleshooting Masalah DNS dan SSL (Pelajaran dari Debugging)

Jika Anda mengalami masalah koneksi, DNS, atau SSL dengan Lighthouse:

1.  **Konfigurasi `systemd-resolved` untuk DNS yang Andal:**
    Jika `nslookup nama_domain_target` (yang menggunakan resolver sistem di `127.0.0.53`) gagal dengan `NXDOMAIN` atau error serupa:
    a.  Edit file konfigurasi `systemd-resolved`:
        ```bash
        sudo nano /etc/systemd/resolved.conf
        ```
    b.  Di bawah bagian `[Resolve]`, uncomment atau tambahkan baris berikut:
        ```ini
        DNS=8.8.8.8 1.1.1.1
        FallbackDNS=8.8.4.4 1.0.0.1
        Domains=~.
        ```
    c.  Simpan file, lalu restart layanan `systemd-resolved`:
        ```bash
        sudo systemctl restart systemd-resolved.service
        ```
    d.  Verifikasi DNS lagi dengan `nslookup nama_domain_target`. Seharusnya sekarang berhasil.

2.  **Tes Koneksi dan Verifikasi SSL Tingkat Sistem:**
    Gunakan `openssl s_client` untuk mengetes koneksi SSL ke endpoint HTTPS secara langsung dan memverifikasi sertifikatnya dari sudut pandang sistem operasi:
    ```bash
    openssl s_client -connect HOST_TARGET:443 -servername HOST_TARGET
    ```
    Contoh: `openssl s_client -connect sepolia.checkpoint.sigp.io:443 -servername sepolia.checkpoint.sigp.io`
    Perhatikan outputnya, terutama `Verify return code:`. `0 (ok)` berarti sertifikat berhasil diverifikasi oleh sistem Anda. Jika ada error, ini menunjukkan masalah pada CA certificates atau validasi SSL di level OS.

3.  **Pastikan CA Certificates Terbaru:**
    ```bash
    sudo apt update && sudo apt install -y ca-certificates && sudo update-ca-certificates --fresh
    ```

4.  **Pastikan Waktu Sistem Akurat:**
    ```bash
    timedatectl status
    ```
    Pastikan `System clock synchronized: yes`. Jika tidak, perbaiki sinkronisasi waktu (misalnya dengan `sudo systemctl restart systemd-timesyncd.service` jika ada, atau `ntp`, `chrony`).

5.  **Masalah URL Checkpoint Sync Spesifik:**
    Dalam proses debugging, ditemukan bahwa URL `https://sepolia.checkpoint.sigp.io/` menyebabkan error SSL "hostname mismatch" spesifik pada Lighthouse, meskipun `openssl s_client` berhasil. Mengganti ke `https://beaconstate-sepolia.chainsafe.io/` menyelesaikan masalah ini untuk Lighthouse. Selalu tes resolusi DNS (`nslookup`) dan SSL (`openssl s_client`) untuk URL checkpoint baru sebelum digunakan.

## Langkah 6: Memulai dan Memantau Node

1.  **Mulai Layanan (Urutan Penting):**
    a.  Mulai Geth terlebih dahulu:
        ```bash
        sudo systemctl start geth-sepolia.service
        ```
    b.  Tunggu sekitar 1-2 menit agar Geth siap dengan Engine API-nya.
    c.  Mulai Lighthouse:
        ```bash
        sudo systemctl start lighthouse-sepolia-beacon.service
        ```

2.  **Periksa Status Layanan:**
    ```bash
    sudo systemctl status geth-sepolia.service
    sudo systemctl status lighthouse-sepolia-beacon.service
    ```
    Keduanya seharusnya `active (running)`.

3.  **Lihat Log untuk Memantau Sinkronisasi:**
    * Log Geth: `sudo journalctl -fu geth-sepolia.service`
    * Log Lighthouse: `sudo journalctl -fu lighthouse-sepolia-beacon.service`

    **Apa yang Diharapkan di Log:**
    * **Lighthouse:** Akan memulai "checkpoint sync", mencari peer, dan akhirnya melaporkan "Synced" dengan slot, epoch, dan jumlah peer yang meningkat.
    * **Geth:** Akan menunjukkan pesan "Forkchoice requested sync to new head" (menandakan komunikasi dengan Lighthouse), kemudian "Syncing beacon headers", diikuti oleh "Syncing: chain download in progress" dan yang paling lama "Syncing: state download in progress". Perhatikan persentase (`synced=%`) dan estimasi waktu (`eta=`).

    **Kesabaran:** Sinkronisasi state Geth bisa memakan waktu beberapa jam hingga lebih. Lighthouse biasanya lebih cepat jika menggunakan checkpoint sync.

## Langkah 7: Mengakses RPC Node

Setelah kedua klien sepenuhnya tersinkronisasi:
* **Geth HTTP RPC Endpoint:** `http://ALAMAT_IP_VPS_ANDA:8545`
* **Geth WebSocket RPC Endpoint:** `ws://ALAMAT_IP_VPS_ANDA:8546`

Anda bisa mengetes koneksi RPC secara lokal di VPS (jika `curl` terinstal):
```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' [http://127.0.0.1:8545](http://127.0.0.1:8545)
