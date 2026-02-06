# Cloud Platforms 2025–2026 — Smart Campus Window Monitoring (Event‑Driven Pipeline)

Containerized, event‑driven σύστημα επιτήρησης αισθητήρων παραθύρων (Smart Campus).
Στόχος: real‑time δημιουργία/εκκαθάριση alarms για “ανοιχτό παράθυρο” και κονσόλα φύλακα (dashboard + acknowledge),
με πλήρη αυτοματοποιημένη διασύνδεση μεταξύ cloud υπηρεσιών.

---

## 1) Τεχνολογίες που χρησιμοποιούνται (>=4 από τη λίστα)

Η υλοποίηση αξιοποιεί τις παρακάτω τεχνολογίες:

- **Node‑RED**: συλλογή + κανονικοποίηση telemetry από Smart Campus ThingsBoard (polling) και αποθήκευση σε MinIO.
- **MinIO (Object Storage / S3)**: αρχειοθέτηση κάθε μέτρησης ως object + metadata.
- **RabbitMQ (Messaging / AMQP)**: message broker για events από MinIO και buffering προς consumer.
- **ThingsBoard (IoT Platform)**: κατανάλωση events, mapping σε devices, Rule Chain για create/clear alarms & battery threshold.
- **Docker / docker‑compose**: πλήρως containerized deployment με αυτόματη παραμετροποίηση μέσω env vars.

> Προαιρετικό/bonus (για μεγαλύτερη βαθμολογία): MicroK8s (Kubernetes) deployment, βλέπε §8.

---

## 2) Σενάριο & Use Cases (για live demo)

### UseCase1 — Άνοιγμα παραθύρου & alarm
1. Έρχεται telemetry με `magnet_status = 1`
2. Δημιουργείται alarm “Window Open”
3. Εμφανίζεται στο dashboard με ειδοποίηση
4. Ο φύλακας κάνει **Acknowledge** (ανάληψη συμβάντος)

### UseCase2 — Κλείσιμο παραθύρου & αυτόματο clear
1. Έρχεται telemetry με `magnet_status = 0`
2. Γίνεται **Clear** το αντίστοιχο alarm
3. Το dashboard ενημερώνεται σε πραγματικό χρόνο

---

## 3) Αρχιτεκτονική (End‑to‑End data flow)

```text
Smart Campus ThingsBoard (source) 
   │  (HTTP API, JWT)
   ▼
Node‑RED (poll, normalize, put object)
   │  (S3 put)
   ▼
MinIO (store payload + metadata)
   │  (Bucket Notifications via AMQP)
   ▼
RabbitMQ (queue/buffer)
   │  (consumer)
   ▼
ThingsBoard (integration + rule chain)
   │
   ▼
Guard Dashboard (alarms table + acknowledge)
```

**Σημείο-κλειδί (automation/integration):**
Κάθε `PUT object` στο MinIO παράγει **event** (bucket notification) που πάει αυτόματα στο RabbitMQ,
χωρίς polling από scripts.

---

## 4) Repo / Components (monorepo λογική)

Ενδεικτικά components που τρέχουν στο stack:

- `nodered` (flows + settings)
- `minio` (object storage + bucket notifications)
- `rabbitmq` (broker)
- `thingsboard` (+ db αν χρησιμοποιείται)
- `docker-compose.yml` για εκκίνηση όλων μαζί

---

## 5) Προαπαιτούμενα

- Docker & Docker Compose (v2)
- (Προαιρετικά) VM + MicroK8s για Kubernetes deployment (§8)
- Smart Campus **reader account** (αν τραβάς live δεδομένα από production)

---

## 6) Quick Start (Docker Compose)

### 6.1 Clone
```bash
git clone https://github.com/giannismavr/cloud_platforms26.git
cd cloud_platforms26
```

### 6.2 Environment variables
Δημιούργησε `.env` (ή χρησιμοποίησε ό,τι προβλέπει το repo) και συμπλήρωσε κωδικούς.

Παράδειγμα `.env` (προσαρμόζεις ανάλογα με το compose):
```env
# --- MinIO ---
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin

# --- RabbitMQ ---
RABBITMQ_DEFAULT_USER=guest
RABBITMQ_DEFAULT_PASS=guest

# --- ThingsBoard (local) ---
TB_HOST=http://thingsboard:8080
TB_SYSADMIN_EMAIL=sysadmin@thingsboard.org
TB_SYSADMIN_PASSWORD=sysadmin

# --- Smart Campus ThingsBoard (source) ---
SMARTCAMPUS_TB_BASEURL=https://<smartcampus-host>
SMARTCAMPUS_USER=<reader-email>
SMARTCAMPUS_PASS=<reader-password>

# --- Node-RED polling ---
POLL_INTERVAL_SECONDS=50
```

> Σημείωση: Τα Smart Campus credentials **δεν** τα κάνουμε commit. Μπαίνουν μόνο σε `.env` ή secrets.

### 6.3 Start
```bash
docker compose up -d --build
docker compose ps
```

### 6.4 Logs
```bash
docker compose logs -f
```

---

## 7) Accounts / Endpoints για δοκιμή (README requirement)

> Τα ακριβή ports/URLs εξαρτώνται από το `docker-compose.yml`. Δες τα με `docker compose ps`.

### 7.1 Node‑RED
- URL: `http://<host>:<port>`
- (Αν υπάρχει auth) χρησιμοποίησε τα credentials που ορίζονται στο compose ή σε `.env`.

### 7.2 MinIO
- Console: `http://<host>:<port>`
- Access Key / Secret Key: από `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`

### 7.3 RabbitMQ
- Management UI: `http://<host>:<port>`
- User/Pass: από `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`

### 7.4 ThingsBoard (local instance)
- URL: `http://<host>:<port>`
- Sysadmin: από `TB_SYSADMIN_EMAIL`, `TB_SYSADMIN_PASSWORD`

### 7.5 Smart Campus (data source)
Για πρόσβαση σε live δεδομένα απαιτείται reader account (βλ. εκφώνηση / οδηγίες μαθήματος).

---

## 8) (Bonus) Kubernetes Deployment (MicroK8s)

> Αν θες να “πιάσεις” και το κριτήριο Kubernetes: τρέχεις το ίδιο σύστημα σε MicroK8s VM.

### 8.1 Setup MicroK8s
```bash
sudo snap install microk8s --classic
microk8s status --wait-ready
microk8s enable dns storage
alias kubectl='microk8s kubectl'
```

### 8.2 Build & push images (Docker Hub / GHCR)
Παράδειγμα (προσαρμόζεις τα ονόματα):
```bash
docker build -t <dockerhub_user>/cloud-nodered:latest ./nodered
docker push <dockerhub_user>/cloud-nodered:latest
```

Κάνε αντίστοιχα για minio / rabbitmq / thingsboard (αν υπάρχουν custom Dockerfiles).

### 8.3 Deploy
Δύο επιλογές:
- **Option A (recommended):** γράφεις manifests (`k8s/`) και κάνεις `kubectl apply -f k8s/`
- **Option B:** χρησιμοποιείς `kompose` για μετατροπή compose→k8s

Παράδειγμα Option B:
```bash
sudo snap install kompose
kompose convert -f docker-compose.yml -o k8s/
kubectl apply -f k8s/
kubectl get pods -A
```

---

## 9) Demo checklist (15’ παρουσίαση)

- `docker compose up -d`
- Δείχνω:
  1) Node‑RED flow που τρέχει ανά ~50s
  2) MinIO bucket με objects + metadata
  3) MinIO bucket notifications (AMQP target)
  4) RabbitMQ queue (messages/rates)
  5) ThingsBoard integration + Rule Chain (create/clear alarm, battery threshold)
  6) Guard Dashboard: alarms + acknowledge
- Εκτελώ UseCase1 και UseCase2 live (ή video demo αν απαιτηθεί)

---

## 10) Troubleshooting

- **Δεν εμφανίζονται alarms**
  - Έλεγξε ότι ο Node‑RED κάνει successful login (JWT) και γράφει objects στο MinIO.
  - Έλεγξε ότι το MinIO στέλνει notifications προς RabbitMQ.
  - Έλεγξε ότι το ThingsBoard καταναλώνει από τη σωστή queue/integration.

- **Alarm δεν κάνει clear**
  - Το clear γίνεται όταν έρθει νέο telemetry με `magnet_status=0` και περάσει από Rule Chain.

---

## 11) Authors / Team

- Παναγιώτης Χάρος (ais25133)
- Ιωάννης Μαυροδήμος (ais25126)
