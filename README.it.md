
[🇬🇧 English](README.md) | 🇮🇹 Italiano

# Diario di Laboratorio — AI Local con llama.cpp e Docker

> **Questo documento è il mio diario tecnico personale.**
> Lo scrivo per me stesso, in modo da poter ripercorrere ogni passo del progetto. È scritto in italiano, con spiegazioni in linea per ogni termine o procedura che potrebbe non essere immediata. Se stai leggendo questo su GitHub, benvenuto: questo è un resoconto  di come ho installato e configurato un server LLM locale con accelerazione GPU su Kali Linux, inclusi tutti gli errori che ho incontrato e come li ho risolti.

---

## Indice

1. [Obiettivo del progetto](#1-obiettivo-del-progetto)
2. [Hardware e software utilizzati](#2-hardware-e-software-utilizzati)
3. [Struttura del progetto](#3-struttura-del-progetto)
4. [Fase 1 — Installazione di Docker](#4-fase-1--installazione-di-docker)
5. [Fase 2 — Configurazione permessi Docker](#5-fase-2--configurazione-permessi-docker)
6. [Fase 3 — Installazione NVIDIA Container Toolkit](#6-fase-3--installazione-nvidia-container-toolkit)
7. [Fase 4 — Aggiornamento driver NVIDIA](#7-fase-4--aggiornamento-driver-nvidia)
8. [Fase 5 — Configurazione dei file del progetto](#8-fase-5--configurazione-dei-file-del-progetto)
9. [Fase 6 — Avvio del server](#9-fase-6--avvio-del-server)
10. [Errori incontrati e soluzioni](#10-errori-incontrati-e-soluzioni)
11. [Comandi di uso quotidiano](#11-comandi-di-uso-quotidiano)
12. [Note finali e lezioni apprese](#12-note-finali-e-lezioni-apprese)

---

## 1. Obiettivo del progetto

L'obiettivo è eseguire un modello LLM (Large Language Model — un modello di intelligenza artificiale per il linguaggio, come quelli alla base di ChatGPT) in locale sul mio computer, sfruttando la GPU NVIDIA per accelerare l'inferenza (il processo con cui il modello genera le risposte).

Il software scelto è **llama.cpp**, un motore di inferenza open-source che supporta modelli in formato GGUF (un formato compresso ottimizzato per l'esecuzione locale). llama.cpp viene eseguito all'interno di un **container Docker** (un ambiente isolato e riproducibile, simile a una macchina virtuale leggera) per semplificarne la gestione.

Il risultato finale è un server HTTP accessibile su `http://localhost:8081` che espone un'API usabile da qualsiasi client (browser, app, script Python).

---

## 2. Hardware e software utilizzati

| Componente | Dettaglio |
|---|---|
| CPU | Intel i5-7500 |
| RAM | 16 GB |
| GPU | NVIDIA GeForce RTX 3060 (12 GB VRAM) |
| Storage | NVMe 465 GB |
| Sistema operativo | Kali Linux |
| Driver NVIDIA | 595.80 |
| CUDA | 13.2 |
| Docker | 28.5.2 |
| llama.cpp | immagine `ghcr.io/ggml-org/llama.cpp:full-cuda` |
| Modelli utilizzati | Gemma-4 E4B (Q4_K_XL), Dolphin Llama3.1 8B (Q8_0) |

> **CUDA** è una piattaforma sviluppata da NVIDIA che permette ai programmi di usare la GPU per calcoli generici (non solo grafica). La versione di CUDA dipende dal driver installato: driver più recenti supportano versioni CUDA più alte.

---

## 3. Struttura del progetto

```
~/Progetti/AI-local/
├── docker-compose.yml   # Definisce come avviare il container
├── models.ini           # Configurazione dei modelli LLM
└── gguf/
    ├── gemma-4-E4B-it-UD-Q4_K_XL.gguf
    └── Dolphin3.0-Llama3.1-8B-Q8_0.gguf
```

Creare la struttura da zero:

```bash
mkdir -p ~/Progetti/AI-local/gguf
cd ~/Progetti/AI-local
```

> `mkdir -p` crea la cartella e tutte le cartelle intermedie necessarie. `-p` sta per "parents".

---

## 4. Fase 1 — Installazione di Docker

### Problema iniziale

Docker non era installato. Il sistema aveva solo `docker-cli` disponibile nei repo, ma questo pacchetto contiene solo il client (lo strumento che invia comandi) senza il daemon (il processo che esegue i container). Senza il daemon, Docker non funziona.

### Soluzione

Installare `docker.io`, che include tutto il necessario:

```bash
sudo apt update
sudo apt install docker.io
```

> `apt` è il gestore di pacchetti di Debian/Kali. `sudo` esegue il comando come amministratore. `docker.io` è il pacchetto completo che include il daemon `dockerd`, il client, e `containerd` (il runtime sottostante).

Avviare e verificare il daemon:

```bash
sudo systemctl start docker
sudo systemctl status docker
```

> `systemctl` è lo strumento per gestire i servizi di sistema in Linux. `start` avvia il servizio, `status` ne mostra lo stato. Il daemon Docker si chiama `docker.service`.

Output atteso da `status`:
```
Active: active (running)
```

Abilitare Docker all'avvio automatico (opzionale ma consigliato):

```bash
sudo systemctl enable docker
```

---

## 5. Fase 2 — Configurazione permessi Docker

### Problema

Eseguendo `docker-compose up -d` senza privilegi, compariva questo errore:

```
permission denied while trying to connect to the Docker daemon socket
at unix:///var/run/docker.sock
```

### Spiegazione

Il daemon Docker comunica attraverso un file speciale chiamato **socket** (`/var/run/docker.sock`). Per default, solo l'utente `root` e gli utenti nel gruppo `docker` possono accedervi. L'utente `kali` non era in quel gruppo.

### Soluzione

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> `usermod` modifica le impostazioni di un utente. `-a` significa "aggiungi" (senza togliere i gruppi esistenti). `-G docker` specifica il gruppo da aggiungere. `$USER` è una variabile d'ambiente che contiene automaticamente il nome dell'utente corrente.
>
> `newgrp docker` apre una nuova shell con il gruppo `docker` già attivo, senza dover fare logout. Equivale a riloggare, ma più rapido. **Nota:** va eseguito ogni volta che si apre un nuovo terminale, finché non si fa un logout completo e poi login.

Verifica che il gruppo sia attivo:

```bash
groups
```

Nell'output deve comparire `docker`.

---

## 6. Fase 3 — Installazione NVIDIA Container Toolkit

### Problema

Anche con Docker funzionante, avviare il container con supporto GPU produceva:

```
could not select device driver "nvidia" with capabilities: [[gpu]]
```

### Spiegazione

Docker da solo non sa come parlare con la GPU NVIDIA. Serve il **NVIDIA Container Toolkit**, un layer software che fa da ponte tra Docker e il driver NVIDIA installato sul sistema.

### Soluzione

Aggiungere il repository ufficiale NVIDIA e installare il toolkit:

```bash
# Scaricare e salvare la chiave GPG del repository NVIDIA
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# Aggiungere il repository alla lista di sorgenti di apt
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Installare
sudo apt update && sudo apt install -y nvidia-container-toolkit
```

> **Chiave GPG**: una firma crittografica che garantisce che i pacchetti scaricati provengano davvero da NVIDIA e non siano stati manomessi.
>
> `curl` è uno strumento per scaricare contenuti da URL. `-fsSL` significa: fallisci silenziosamente in caso di errore (`f`), non mostrare progress bar (`s`), segui i redirect (`L`).
>
> `gpg --dearmor` converte la chiave dal formato testo al formato binario usato da apt.
>
> `tee` scrive l'output sia su file che sullo schermo contemporaneamente.
>
> `sed` è uno strumento per modificare testo. Qui aggiunge il riferimento alla chiave GPG dentro il file del repository.

Registrare il runtime NVIDIA in Docker e riavviare:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

> `nvidia-ctk` è lo strumento di configurazione del toolkit. `runtime configure` modifica `/etc/docker/daemon.json` aggiungendo il runtime `nvidia`. Senza questo passaggio, Docker non sa che il runtime esiste.

---

## 7. Fase 4 — Aggiornamento driver NVIDIA

### Problema

Anche dopo aver installato il toolkit, il container non partiva con:

```
nvidia-container-cli: requirement error: unsatisfied condition: cuda>=12.8,
please update your driver to a newer version
```

### Spiegazione

L'immagine Docker `ghcr.io/ggml-org/llama.cpp:full-cuda` è compilata contro CUDA 12.8+. Il driver installato (versione 550, CUDA 12.4) era troppo vecchio. I repository di Kali Linux avevano solo il driver 550, quindi non era possibile aggiornarlo tramite `apt`.

### Verifica della situazione

```bash
nvidia-smi                          # mostra versione driver e CUDA attuale
apt-cache policy nvidia-driver      # mostra versione disponibile nei repo
apt-cache search nvidia-driver | grep -v lib  # elenca i pacchetti disponibili
```

### Soluzione: installazione manuale del driver

Scaricare il driver più recente dal sito ufficiale NVIDIA:
- URL: https://www.nvidia.com/en-us/drivers/
- Selezionare: GeForce → GeForce RTX 30 Series → RTX 3060 → Linux 64-bit
- Versione scaricata: **595.80** (rilasciata il 27 maggio 2026)

Fermare il server grafico (X11) prima dell'installazione, perché il driver non può essere installato mentre la GPU è in uso dalla grafica:

```bash
sudo systemctl stop display-manager
```

> `display-manager` è il servizio che gestisce l'interfaccia grafica (login screen e desktop). Fermarlo porta il sistema in modalità console testuale (schermo nero con cursore lampeggiante — comportamento normale).

Eseguire l'installer:

```bash
sudo sh ~/Scaricati/NVIDIA-Linux-x86_64-595.80.run
```

Durante l'installazione, il wizard interattivo ha presentato queste scelte:

| Domanda | Risposta | Motivazione |
|---|---|---|
| Tipo di kernel module | **MIT/GPL** | Modulo open-source, più compatibile con kernel Linux moderni. Consigliato per RTX 30 series in avanti. |
| Disabilitare Nouveau | **Yes** | Nouveau è il driver open-source generico per GPU NVIDIA. Entra in conflitto con il driver proprietario e va disabilitato. |
| Ricostruire initramfs | **Rebuild initramfs** | L'initramfs è l'immagine caricata dal kernel all'avvio. Va ricostruita per includere la disabilitazione di Nouveau, altrimenti al riavvio Nouveau si ricarica. |
| Installare librerie 32-bit | **Yes** | Utili per applicazioni legacy (es. Steam). Non pesano molto. |
| Aggiornare configurazione X11 | **Yes** | Aggiorna il file di configurazione del server grafico per usare il driver NVIDIA al prossimo avvio. |

Riavviare il sistema:

```bash
sudo reboot
```

Verificare l'installazione:

```bash
nvidia-smi
```

Output atteso: driver 595.80, CUDA Version 13.2.

> **Attenzione:** il driver installato con questo metodo è fuori dalla gestione di `apt`. Dopo aggiornamenti del kernel Linux potrebbe essere necessario reinstallarlo con lo stesso file `.run`.

---

## 8. Fase 5 — Configurazione dei file del progetto

### docker-compose.yml

Il file `docker-compose.yml` definisce come Docker deve avviare il container. Crearlo in `~/Progetti/AI-local/`:

```yaml
services:
  llama-server:
    image: ghcr.io/ggml-org/llama.cpp:full-cuda
    container_name: llama-server
    ports:
      - "8081:8081"
    volumes:
      - /home/kali/Progetti/AI-local/gguf/:/models
      - ./models.ini:/config.ini
    command: |
      --server
      --models-preset /config.ini
      --sleep-idle-seconds 600
      --parallel 1
      --port 8081
      --host 127.0.0.1
      --jinja
      --models-autoload
      --models-max 1
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: on-failure
```

**Spiegazione dei campi principali:**

| Campo | Significato |
|---|---|
| `image` | L'immagine Docker da usare. `full-cuda` include llama.cpp compilato con supporto CUDA. |
| `ports: "8081:8081"` | Mappa la porta 8081 del container alla porta 8081 del computer host. Formato: `host:container`. |
| `volumes` | Monta cartelle del computer host dentro il container. La cartella `gguf/` diventa `/models` dentro il container. |
| `--models-preset /config.ini` | Usa il file models.ini per caricare i preset dei modelli. |
| `--models-autoload` | Carica il modello in memoria solo quando arriva la prima richiesta (lazy loading). |
| `--models-max 1` | Tiene al massimo un modello caricato in VRAM alla volta. |
| `--sleep-idle-seconds 600` | Scarica il modello dalla VRAM dopo 10 minuti di inattività. |
| `capabilities: [gpu]` | Dichiara che il container ha bisogno dell'accesso alla GPU. |
| `restart: on-failure` | Riavvia il container automaticamente se crasha. |

### models.ini

Il file `models.ini` configura i modelli disponibili. Crearlo in `~/Progetti/AI-local/`:

```ini
[*] ; Parametri globali, validi per tutti i modelli
flash-attn = on
gpu-layers = 999
batch-size = 512
ubatch-size = 256
cache-type-k = q4_0
cache-type-v = q4_0
repeat-penalty = 1.0
cont-batching = true
slots = 1
fit = on

[Gemma-4 E4B] ; Preset specifico per Gemma
model = /models/gemma-4-E4B-it-UD-Q4_K_XL.gguf
c = 16384
temp = 0.8

[Dolphin Llama3.1 8B] ; Preset specifico per Dolphin
model = /models/Dolphin3.0-Llama3.1-8B-Q8_0.gguf
c = 16384
temp = 0.8
```

**Spiegazione dei parametri principali:**

| Parametro | Significato |
|---|---|
| `gpu-layers = 999` | Carica tutti i layer del modello in VRAM. Con 999 si intende "tutti quelli possibili". |
| `flash-attn = on` | Abilita Flash Attention, un algoritmo ottimizzato che riduce l'uso di VRAM durante l'inferenza. |
| `cache-type-k/v = q4_0` | Quantizza la KV cache (memoria usata per il contesto) in 4-bit per risparmiare VRAM. |
| `c = 16384` | Context size: quanti token (unità di testo, circa 0.75 parole in media) il modello può tenere in memoria durante una conversazione. 8192 è il minimo, 32768 è il massimo ragionevole con 12GB VRAM. |
| `temp = 0.8` | Temperatura: controlla la "creatività" delle risposte. 0 = deterministico, 1 = molto vario. |
| `fit = on` | Se il modello non entra interamente in VRAM, carica automaticamente i layer rimanenti in RAM. |

---

## 9. Fase 6 — Avvio del server

### Prima volta (scarica l'immagine Docker)

```bash
cd ~/Progetti/AI-local
newgrp docker
docker-compose up -d
```

La prima esecuzione scarica l'immagine `full-cuda` da GitHub Container Registry (~15 GB). Richiede tempo dipendente dalla connessione.

> `docker-compose up` avvia i servizi definiti nel `docker-compose.yml`. `-d` sta per "detached": il container gira in background e il terminale rimane libero.

### Verificare che il server sia attivo

```bash
docker ps
```

Deve mostrare `llama-server` con status `Up`.

```bash
docker logs llama-server
```

Output atteso:
```
srv  llama_server: router server is listening on http://127.0.0.1:8081
```

### Monitorare l'uso della GPU

```bash
watch -n 1 nvidia-smi
```

> `watch -n 1` riesegue il comando ogni secondo e aggiorna lo schermo. Utile per vedere in tempo reale quanta VRAM viene usata durante l'inferenza. Uscire con `Ctrl+C`.

Con il modello Gemma-4 E4B caricato, l'utilizzo atteso è circa 3600-4000 MiB su 12288 MiB totali.

### Accedere al server

Aprire il browser e andare su:
```
http://localhost:8081
```

L'interfaccia web di llama.cpp permette di chattare direttamente con il modello.

---

## 10. Errori incontrati e soluzioni

### Errore 1 — `docker.service not found`

**Messaggio:**
```
Failed to start docker.service: Unit docker.service not found.
```

**Causa:** era installato solo `docker-cli`, non il daemon completo.

**Soluzione:**
```bash
sudo apt install docker.io
sudo systemctl start docker
```

---

### Errore 2 — `permission denied` sul socket Docker

**Messaggio:**
```
permission denied while trying to connect to the Docker daemon socket
at unix:///var/run/docker.sock
```

**Causa:** l'utente non era nel gruppo `docker`.

**Soluzione:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

### Errore 3 — `could not select device driver "nvidia"`

**Messaggio:**
```
could not select device driver "nvidia" with capabilities: [[gpu]]
```

**Causa:** NVIDIA Container Toolkit non installato o non configurato.

**Soluzione:**
```bash
# Installare il toolkit (vedere Fase 3)
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
newgrp docker
```

---

### Errore 4 — `cuda>=12.8` non soddisfatto

**Messaggio:**
```
nvidia-container-cli: requirement error: unsatisfied condition: cuda>=12.8
```

**Causa:** driver NVIDIA versione 550 (CUDA 12.4), troppo vecchio per l'immagine Docker corrente.

**Soluzione:** aggiornare il driver alla versione 595.80 tramite installer ufficiale (vedere Fase 4).

---

### Errore 5 — context size superato

**Messaggio:**
```
request (10821 tokens) exceeds the available context size (8192 tokens)
```

**Causa:** il parametro `c` in `models.ini` era impostato a 8192, insufficiente per conversazioni lunghe.

**Soluzione:** aumentare il context size in `models.ini`:
```ini
c = 16384
```
Poi riavviare:
```bash
docker-compose restart
```

---

### Errore 6 — nome file modello errato

**Causa:** in `models.ini` era scritto `Q5_K_XL` ma il file reale era `Q4_K_XL`.

**Soluzione:** verificare il nome esatto del file:
```bash
ls ~/Progetti/AI-local/gguf/
```
Correggere `models.ini` con il nome esatto.

---

## 11. Comandi di uso quotidiano

### Avviare il server

```bash
cd ~/Progetti/AI-local
newgrp docker
docker-compose up -d
```

### Fermare il server

```bash
cd ~/Progetti/AI-local
docker-compose down
```

### Riavviare dopo modifiche a models.ini

```bash
docker-compose restart
```

### Vedere i log in tempo reale

```bash
docker logs -f llama-server
```

> `-f` sta per "follow": i log si aggiornano in tempo reale. Uscire con `Ctrl+C`.

### Monitorare GPU

```bash
watch -n 1 nvidia-smi
```

### Verificare i container attivi

```bash
docker ps
```

---

## 12. Note finali e lezioni apprese

**Memoria tra sessioni:** il modello non ha memoria persistente. Ogni nuova chat riparte da zero. Per progetti in sviluppo, mantenere un file `context.md` con il riassunto del progetto da incollare all'inizio di ogni nuova sessione.

**Context size e VRAM:** con 12 GB di VRAM e il modello Gemma-4 E4B Q4, il context size massimo ragionevole è circa 32768 token. Valori più alti causano out-of-memory. Monitorare con `nvidia-smi` durante l'inferenza.

**Driver NVIDIA fuori da apt:** il driver 595.80 è installato manualmente. Dopo aggiornamenti del kernel Linux, potrebbe essere necessario reinstallarlo con:
```bash
sudo systemctl stop display-manager
sudo sh ~/Scaricati/NVIDIA-Linux-x86_64-595.80.run
sudo reboot
```

**newgrp docker:** va eseguito ogni volta che si apre un nuovo terminale, finché non si fa un logout completo. Dopo il logout e il login successivo, il gruppo è permanente e `newgrp` non serve più.

**Modelli disponibili:**
- `Gemma-4 E4B Q4_K_XL` — modello compatto, buon rapporto qualità/velocità, ottimizzato per italiano
- `Dolphin Llama3.1 8B Q8_0` — modello più grande, qualità superiore, usa più VRAM

Con `--models-max 1`, solo un modello alla volta è caricato in VRAM. Cambiando modello nella UI, il precedente viene scaricato automaticamente.
