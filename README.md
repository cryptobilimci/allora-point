# Allora Teşvikli Testnet - Phase 2

## Allora total 34M$ fonlandı.
![image](https://github.com/papaperez1/Allora/assets/118633093/1b0aa7b8-16f0-4862-a9d8-26c0b2689c7c)

>İşlemler için sanal sunucu alıp yapabilirsiniz.

### Min Sistem Gereksinim
CPU: Minimum 1/2 core.
RAM: 2 to 4 GB.
Depolama: SSD or NVMe with at least 5GB of space.
OS :UBUNTU 22.04 LTS

---------------------------------------------

### Sistem Güncellemesi
```
sudo apt update
sudo apt upgrade -y
```

### GO Yükle
```
curl -OL https://go.dev/dl/go1.22.4.linux-amd64.tar.gz
```
```
tar -C /usr/local -xvf go1.22.4.linux-amd64.tar.gz
```
```
export PATH=$PATH:/usr/local/go/bin
```
```
go version
```

### Python3 Yüklemesi
```
apt install python3-pip
```
### Docker Yüklemesi
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
```
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Docker Güncelleme
```
sudo curl -L "https://github.com/docker/compose/releases/download/2.28.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
```
sudo chmod +x /usr/local/bin/docker-compose
```
```
docker compose version
```

### CLI Yüklemesi
```
pip install allocmd --upgrade
```
```
allocmd --version
```
----------------------------------
## Kısım 2 : Cüzdan Kurulum ve Worker Çalıştırma

### Cüzdan Kurulum
```
curl -sSL https://raw.githubusercontent.com/allora-network/allora-chain/main/install.sh | bash -s -- v0.0.10
```
```
export PATH="$PATH:/root/.local/bin"
```
```
git clone -b v0.0.10 https://github.com/allora-network/allora-chain.git
```
```
cd allora-chain && make all
```
```
allorad version
```

#### Yeni Cüzdan Oluşturma 
```
allorad keys add YENI_CUZDAN_ISMI
```

> Cüzdanı oluşturduktan sonra allo ile başlayan cüzdan adresinizle birlikte 24 kelimelik cüzdan seed word'lerinide verecek.
>
> Onları bir yere not edin!

### Prediction Worker Kurulumu
```
cd $HOME && git clone https://github.com/allora-network/basic-coin-prediction-node && cd basic-coin-prediction-node
```
```
mkdir worker-data
mkdir head-data
```
```
sudo chmod -R 777 worker-data
sudo chmod -R 777 head-data
```

#### Head key oluşturma
```
sudo docker run -it --entrypoint=bash -v ./head-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
```

#### Worker key oluşturma
```
sudo docker run -it --entrypoint=bash -v ./worker-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
```

> Head Key'i aşağıdaki kodla alın ve bir yere not edin.
```
cat head-data/keys/identity
```
![image](https://github.com/papaperez1/Allora/assets/118633093/0fe235f7-18d1-458f-9b46-c334583be6ad)

----------------------------
## Worker Çalıştırma

```
rm -rf docker-compose.yml && nano docker-compose.yml
```
> Yukarıdaki dosya konumuna gidip `HEAD-KEY` & `CUZDAN_SEED_KELIME` olan kısımları CTRL+W ile aratıp değiştirin.
>
> `HEAD-KEY` bir önceki komutla aldığınız kod - `CUZDAN_SEED_KELIMEE en başta bize verdiği 24 Kelimelik cüzdan kurtarma kelimeleri.

--------------

```
version: '3'

services:
  inference:
    container_name: inference-basic-eth-pred
    build:
      context: .
    command: python -u /app/app.py
    ports:
      - "8000:8000"
    networks:
      eth-model-local:
        aliases:
          - inference
        ipv4_address: 172.22.0.4
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/inference/ETH"]
      interval: 10s
      timeout: 5s
      retries: 12
    volumes:
      - ./inference-data:/app/data

  updater:
    container_name: updater-basic-eth-pred
    build: .
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
    command: >
      sh -c "
      while true; do
        python -u /app/update_app.py;
        sleep 24h;
      done
      "
    depends_on:
      inference:
        condition: service_healthy
    networks:
      eth-model-local:
        aliases:
          - updater
        ipv4_address: 172.22.0.5

  worker:
    container_name: worker-basic-eth-pred
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9011 \
          --boot-nodes=/ip4/172.22.0.100/tcp/9010/p2p/head-id \
          --topic=1 \
          --allora-chain-key-name=testkey \
          --allora-chain-restore-mnemonic='WALLET_SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.edgenet.allora.network/ \
          --allora-chain-topic-id=1
    volumes:
      - ./worker-data:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker
        ipv4_address: 172.22.0.10

  head:
    container_name: head-basic-eth-pred
    image: alloranetwork/allora-inference-base-head:latest
    environment:
      - HOME=/data
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        allora-node --role=head --peer-db=/data/peerdb --function-db=/data/function-db  \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9010 --rest-api=:6000
    ports:
      - "6000:6000"
    volumes:
      - ./head-data:/data
    working_dir: /data
    networks:
      eth-model-local:
        aliases:
          - head
        ipv4_address: 172.22.0.100


networks:
  eth-model-local:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/24

volumes:
  inference-data:
  worker-data:
  head-data:
```

---------------------------

### Build Compose İşlemi
```
docker compose build
docker compose up -d
```

```
docker ps
```
Yukarıdaki komuttan sonra CONTAİNER ID kısmını kopyalayıp not defterinize kaydedin.
![image](https://github.com/papaperez1/Allora/assets/118633093/c165c1f2-a357-4572-97ba-a545acc08f6e)

```
docker logs -f CONTAINER_ID
```
CONTAINER_ID kısmını bir üstteki komut ile aldığınız ID ile değiştirin ve gönderin.

> İşlemler bitince `Success: register node Tx Hash:=82BF67E2E1247B226B8C5CFCF3E4F41076909ADABF3852C468D087D94BD9FC3B` tx hax vermesi lazım.
>
> Punlarınızı dashboard'da https://app.allora.network?ref=eyJyZWZlcnJlcl9pZCI6ImI4MjQ2ZDkzLWU5YjAtNDVkNi1iODRmLWM2ZWVjMDMwMDYyMSJ9 güncellenmiş şekilde görebilirsiniz.
