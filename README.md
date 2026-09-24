# Staging — DeBrand Studio

Ambiente di staging per il sito istituzionale [https://www.debrandstudio.it/](https://www.debrandstudio.it/).

Raggiungibile pubblicamente all'indirizzo:  
👉 **[https://staging.debrandstudio.it/](https://staging.debrandstudio.it/)**

Questo repository contiene l'infrastruttura Docker Compose, le configurazioni PHP e l'ambiente per il sito WordPress di staging con database MariaDB isolato e dedicato.

---

## Architettura

```
                    Internet (HTTPS)
                           │
                 ┌─────────▼──────────┐
                 │ Nginx Proxy Manager│  (container NPM esterno)
                 │  - Let's Encrypt   │
                 │  - Force SSL       │
                 └─────────┬──────────┘
                           │  rete: debrand_network (esterna)
                 ┌─────────▼──────────┐
                 │  staging_wp :80    │  ← NPM instrada staging.debrandstudio.it qui
                 │  wordpress:7.1.2   │
                 └─────────┬──────────┘
                           │  rete: staging_internal (privata)
                     ┌─────┴─────┐
                     │           │
                 ┌───▼───┐   ┌───▼──────────┐
                 │staging│   │ redis-cache  │  (su debrand_network, DB: 2)
                 │  _db  │   └──────────────┘
                 │12.3.2 │
                 └───────┘
```

### Dettaglio Servizi

| Servizio | Container | Immagine | Rete | Persistenza | Note |
|---|---|---|---|---|---|
| `wordpress` | `staging_wp` | `wordpress:7.1.2` | `staging_internal` + `debrand_network` | `./volume:/var/www/html` | Servito da NPM via porta 80 interna |
| `db` | `staging_db` | `mariadb:12.3.2` | `staging_internal` | `./db-data:/var/lib/mysql` | Database dedicato e isolato |
| `redis-cache` | `redis-cache` | `redis:8.4.0` | `debrand_network` | Container esterno | Database Redis dedicato: 2 |

### Scelte architetturali e sicurezza
- **Isolamento del Database**: `staging_db` è collegato solo alla rete privata `staging_internal`. Non espone porte sull'host e non è raggiungibile da altre applicazioni presenti su `debrand_network`, prevenendo collisioni di porte o accessi non autorizzati.
- **Nessuna porta esposta sull'host**: Tutto il traffico HTTPS in ingresso passa attraverso **Nginx Proxy Manager** (NPM) sulla rete condivisa `debrand_network`.
- **Terminazione SSL & Reverse Proxy**: Nel file `wp-config.php` (e in `docker-compose.yaml`) è gestito l'header `HTTP_X_FORWARDED_PROTO` per garantire il funzionamento corretto sotto HTTPS e prevenire loop di redirect.
- **PHP Personalizzato**: [custom-php.ini](file:///home/ubuntu/staging_wp/custom-php.ini) imposta `memory_limit = 256M`, `upload_max_filesize = 128M` e tempi di esecuzione adeguati.

---

## Struttura del Progetto

```
.
├── docker-compose.yaml    # Definizione dello stack Docker
├── custom-php.ini         # Parametri di configurazione PHP
├── .env.example           # Template delle variabili d'ambiente
├── .env                   # Variabili e credenziali effettive (NON versionato)
├── .gitignore             # File e directory esclusi dal controllo versione
├── README.md              # Documentazione del progetto
├── volume/                # Root di WordPress (/var/www/html - ignorato da git)
└── db-data/               # Dati persistenti MariaDB (ignorato da git)
```

---

## Prerequisiti

- **Docker** e **Docker Compose v2** (`docker compose`).
- Rete Docker esterna `debrand_network` già attiva.
- Container **Nginx Proxy Manager** attivo su `debrand_network` con Host configurato per:
  - **Domain Names**: `staging.debrandstudio.it`
  - **Forward Hostname**: `staging_wp`
  - **Forward Port**: `80`
  - **SSL**: Certificato Let's Encrypt con *Force SSL* e *HTTP/2 Support*.

---

## Guida Rapida / Setup

### 1. Configurazione delle variabili d'ambiente
Copia il file template e imposta password sicure per l'istanza:
```bash
cp .env.example .env
```

Modifica i valori in `.env`:
```ini
MYSQL_ROOT_PASSWORD=la_tua_password_root
MYSQL_DATABASE=staging_wp
MYSQL_USER=staging_user
MYSQL_PASSWORD=la_tua_password_utente
WORDPRESS_TABLE_PREFIX=stg_
```

### 2. Avvio dello stack
Avvia i container in background:
```bash
docker compose up -d
```

### 3. Monitoraggio del primo avvio
WordPress copiera automaticamente i file di base nel volume e configurerà `wp-config.php`:
```bash
docker compose logs -f wordpress
```

---

## Comandi Utili

### Gestione Container
```bash
# Avvio / Arresto
docker compose up -d           # Avvia i container
docker compose down            # Ferma i container (i dati in volume e db-data restano intatti)
docker compose restart         # Riavvia lo stack

# Stato e Log
docker compose ps              # Verifica stato e healthcheck
docker compose logs -f         # Log aggregati in tempo reale
docker compose logs -f wordpress
docker compose logs -f db
```

### Accesso ai Container
```bash
# Shell all'interno del container WordPress
docker compose exec wordpress bash

# Client MariaDB interattivo
docker compose exec db mariadb -u staging_user -p staging_wp
```

### Backup e Ripristino Database
```bash
# Backup veloce del database dedicato
docker compose exec db mariadb-dump -u staging_user -p staging_wp > staging_wp_backup_$(date +%Y%m%d).sql

# Ripristino database da dump SQL
docker compose exec -T db mariadb -u staging_user -p staging_wp < staging_wp_backup.sql
```

---

## Note
- L'installazione di WordPress iniziale può essere completata accedendo direttamente a [https://staging.debrandstudio.it/wp-admin/install.php](https://staging.debrandstudio.it/wp-admin/install.php).
- Per abilitare il caching Redis su WordPress, installare e attivare il plugin **Redis Object Cache** (la connessione a `redis-cache:6379` su database `2` è già preconfigurata nelle costanti WP).
