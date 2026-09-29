# Controllo accessi con tessera e conferma sul telefono

Progetto di **Davide Cesari** e **Alessio Ferrari** · classe 5CI · ITT «G. Marconi» Rovereto · a.s. 2026/27


Sistema che affianca all'impianto RFID esistente (porte dei laboratori e armadietti) un **secondo fattore**: la conferma dell'apertura sul telefono dell'intestatario della tessera, firmata con una passkey (WebAuthn).

> **Stato del progetto:** in sviluppo, Fase 0 (sopralluogo e banco di prova).

---

## Il problema

La tessera RFID non dimostra chi la sta usando. Se viene persa o rubata, apre tutto ciò che la carta permette e nel log l'apertura risulta fatta dal proprietario.

## La soluzione

Dopo il passaggio della tessera, il titolare riceve una notifica sul telefono e conferma l'apertura:

1. il server manda una **sfida casuale** (nonce);
2. il telefono la **firma con la chiave privata** della passkey, dopo lo sblocco locale (volto, impronta o PIN);
3. il server **verifica la firma con la chiave pubblica** registrata.

Il server non riceve mai la chiave privata e non vede alcun dato biometrico: la scuola tratta solo dati ordinari, evitando il problema GDPR del riconoscimento facciale su server.

Il sistema lavora in **modalità passiva**: la porta si apre comunque, l'apertura viene valutata a posteriori (entro 30 secondi) e le anomalie vengono segnalate agli admin.

## Flusso di un'apertura

```
Tessera sul lettore
   └─> ESP32 legge l'UID e lo pubblica via MQTT
        └─> Backend: tessera nota e regole rispettate?
             ├─ no ─> Anomalia: log + notifica agli admin
             └─ sì ─> Push con nonce al titolare
                       ├─ firma valida entro 30 s ─> Apertura confermata (solo log)
                       └─ nessuna firma / «Non sono stato io» ─> Anomalia,
                          tessera «in osservazione», notifica agli admin
```

## Architettura

| Componente | Tecnologia | Ruolo |
|---|---|---|
| Nodo porta | ESP32 + lettore RFID + sensore reed + buzzer/LED | Legge l'UID, rileva l'apertura fisica, dà feedback |
| Broker | Mosquitto, MQTT su TLS (porta 8883) | Comunicazione tra nodi e backend, utente e ACL per nodo |
| Backend | Python, FastAPI, paho-mqtt, py_webauthn | Regole di anomalia, verifica WebAuthn, Web Push, API REST |
| Database | PostgreSQL | Eventi, tessere, credenziali, sfide, anomalie |
| Frontend | Angular (PWA) | Vista personale (passkey, conferma) e dashboard admin |
| Camera (opzionale) | ESP32-CAM | Solo su anomalia, fase futura, fuori dal prototipo |

Topic MQTT principali:

- `porte/<id>/tessera`: UID letto dal nodo
- `porte/<id>/stato`: apertura effettiva (reed)
- `porte/<id>/comando`: LED e buzzer

## Regole di anomalia

| # | Regola | Fase |
|---|---|---|
| 1 | Tessera sconosciuta | 1 |
| 2 | Nessuna conferma entro 30 secondi | 1 |
| 3 | Porta aperta senza tessera, o tenuta aperta oltre un minuto | 1 |
| 4 | Apertura fuori fascia oraria o su porta non permessa | 2 |
| 5 | Tessera smarrita o in osservazione | 2 |
| 6 | Sequenza impossibile (stessa tessera su due porte lontane in meno di un minuto) | 2 |
| 7 | Incrocio con l'orario: il docente risulta in lezione in un'altra aula | 2 |

## Hardware (per porta, circa 25–40 €)

- ESP32 DevKit (o ESP32-S3)
- Lettore RFID: RC522 o PN532 (13,56 MHz), RDM6300 (125 kHz), da scegliere dopo il sopralluogo
- Sensore magnetico reed
- Buzzer e LED
- Scatola stampata in 3D, alimentatore USB 5 V

## Struttura del repository

> Da adattare alla struttura effettiva del repo.

```
.
├── firmware/      # Firmware ESP32 (nodo porta)
├── backend/       # FastAPI, motore di regole, WebAuthn, Web Push
├── frontend/      # PWA Angular (personale + dashboard admin)
├── deploy/        # Docker Compose, Mosquitto, Caddy
├── docs/          # Documentazione tecnica, dossier privacy
└── demo/          # Demo del protocollo di Schnorr (approfondimento)
```

## Avvio rapido (sviluppo)

> Da completare man mano che i componenti prendono forma.

```bash
# Clona il repository
git clone <url-del-repo>
cd <nome-repo>

# Avvia broker, backend e database
docker compose up -d
```

**Nota su HTTPS:** WebAuthn funziona solo su HTTPS con un nome di dominio (non un IP). Per lo sviluppo va bene `localhost`; per provare dal telefono si può usare Tailscale (`tailscale cert`) oppure il dominio della scuola con Caddy e Let's Encrypt. Il nome (RP ID) va scelto una volta sola: cambiarlo invalida tutte le passkey registrate.

## Fasi del progetto

| Fase | Periodo | Risultato |
|---|---|---|
| 0 · Sopralluogo e banco | ottobre–novembre | Nodo funzionante sul banco, l'UID compare nella dashboard |
| 1 · Una porta, modalità passiva | dicembre–febbraio | Nodo su una porta reale, regole 1–3, dashboard eventi e anomalie |
| 2 · Conferma sul telefono | marzo–aprile | PWA con passkey, push, conferma WebAuthn, regole 4–7 |
| 3 · Rifinitura ed esame | maggio | Documentazione, dossier privacy, demo di Schnorr, test con docenti volontari |

Regola di lavoro: ogni fase ha un **ramo Git**, una **demo di 5 minuti** e una **pagina di documentazione**.

## Privacy e conformità

- Nessun dato biometrico, nessuna foto, nessun PIN nel database.
- Dati trattati: UID delle tessere, nome e ruolo del personale, email istituzionale, chiavi pubbliche, log degli accessi, orario.
- Conservazione (proposta, da confermare con il DPO): eventi 90 giorni, anomalie fino alla chiusura più 90 giorni, sfide 24 ore.
- Accesso ai dati solo per gli admin, con doppio fattore.
- Gli armadietti restano nel solo monitoraggio delle regole, senza conferma sul telefono.

## Limiti dichiarati

- La porta si apre comunque (modalità passiva).
- Serve il telefono con la PWA installata.
- La firma non prova la prossimità del telefono alla porta.
- Il nodo camera è fuori dal progetto di quest'anno.

## Approfondimento: Schnorr e prove a conoscenza zero

WebAuthn è un'autenticazione a sfida e risposta con firma digitale, **non** una prova a conoscenza zero. Le ZKP sono trattate a parte con una demo Python del protocollo di identificazione di Schnorr (cartella `demo/`).

## Autori

- Davide Cesari
- Alessio Ferrari

Docente di riferimento: prof. Kevin Zago