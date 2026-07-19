# Hexesoft Bridge Fluidra

Ponte tra l'impianto piscina **AstralPool / Fluidra** (bus **Modbus RTU** dietro
gateway **Modbus TCP**, es. Waveshare RS485-to-ETH) e **Home Assistant**, tramite **MQTT**.
Rende disponibili in Home Assistant — in modo automatico e bidirezionale — pompa di calore,
valvola selettrice, moduli GPIO, luci LumiPlus e qualunque dispositivo Modbus generico
(clorinatori, pompe VS, regolatori pH/Redox, dosatrici) descritto nel file di configurazione.

## Cosa fa

Il bridge si collega al gateway Modbus TCP (porta 502) e:

- **scopre da solo** cosa c'è sul bus RS485: interroga gli indirizzi di fabbrica noti (pompa `9`, valvola `11`, GPIO `26-30`, LumiPlus `48`) e riconosce ogni dispositivo dalla sua identità nel blocco `0x00-0x0C`;
- **pubblica in Home Assistant** le entità di tutti i dispositivi configurati (MQTT Discovery) e rimuove i "fantasmi" delle entità che hai tolto dal file;
- **ascolta i comandi** che arrivano da Home Assistant sul topic `pool_bridge/#` e li traduce in scritture Modbus (`FC 0x06`) verso lo slave giusto;
- **legge ciclicamente** lo stato di ogni dispositivo (default: 30 s), lo confronta con l'ultimo pubblicato, e pubblica solo ciò che è cambiato;
- **si riconnette da solo** sia al gateway Modbus sia al broker MQTT in caso di caduta.

## Funzionalità

- **Discovery automatica e pulizia**: all'avvio pubblica tutte le entità dei dispositivi configurati e rimuove da Home Assistant quelle che hai tolto dal file (niente entità fantasma).
- **Controllo bidirezionale in tempo reale**: ogni cambio di stato letto sul bus diventa uno stato MQTT; ogni comando da Home Assistant diventa una scrittura Modbus.
- **Pompa di Calore**: entità `climate` con modes `off / heat / cool / auto`, preset `silent / smart / powerful`, setpoint e temperatura corrente (se hai fornito gli indirizzi Modbus dedicati), diagnostica dei 32 codici allarme PP**/EE**.
- **Valvola Selettrice SVRAC**: posizione (Filtrazione / Chiusa / Ricircolo), pulsante `Ciclo di lavaggio` (backwash + risciacquo), diagnostica di 14 allarmi. Il pulsante di **SCARICO** compare **solo** se abilitato in configurazione (`allow_drain: true`) — vedi *Sicurezza scarico*.
- **GPIO GP4I-4O**: 4 relè liberamente configurabili come `switch` o `light`, 4 ingressi digitali con `device_class` HA (`problem`, `moisture`, `motion`, …), contatori impulsi opzionali (per il *contatore backwash*, ad esempio).
- **Luci LumiPlus**: on/off, colore predefinito (`select` con i nomi che gli dai), sequenza e velocità (`number`). L'entità è `optimistic` perché il modulo è unidirezionale (telecomando non rilevato).
- **Driver Generico**: qualunque dispositivo Modbus può essere esposto scrivendo la sua mappa registri direttamente nel file di configurazione (`mappings`). Serve solo il manuale Modbus del dispositivo, niente codice da compilare.
- **Log leggibile**: console colorata per livello, con verbosità configurabile (`info`, `debug`, `warning`, `error`), con formato tabellare simmetrico (vedi *Formato del log*).
- **Stato del bridge**: entità Online/Offline (Last Will MQTT), doppia disponibilità Bridge AND Dispositivo (se scolleghi il gateway, le entità di quella pompa diventano non disponibili senza spegnere le altre).

## Come funziona (in breve)

Il bridge apre **una** connessione TCP verso il gateway Modbus e serializza al suo
interno tutte le transazioni con un semaforo. Sul bus RS485 c'è un solo master (noi)
e tanti slave che rispondono a turno: sovrapporre due transazioni significa perdere
entrambe le risposte, quindi serializzare non è un difetto, è il protocollo.

Ogni giro di polling produce lo stato di tutti i registri di tutti i dispositivi
online. Il driver di ciascuno restituisce una lista di aggiornamenti; il bridge li
confronta con l'ultimo pubblicato e manda su MQTT **solo ciò che è cambiato**.

## Sicurezza scarico (valvola SVRAC)

La posizione **SCARICO** della valvola manda l'acqua di piscina **in fogna**. Il
manuale prescrive che l'esecuzione richieda **due passi**: prima un *Request DRAIN*
che apre una finestra di 10 secondi, poi un *Confirm DRAIN* dentro quella finestra.

Il bridge rispetta questa sequenza, ed in più:
- lo scarico è **disabilitato di default** (`allow_drain: false` nella configurazione della valvola);
- quando abilitato, appare **come pulsante separato** e non come voce del menu a tendina delle posizioni, così non lo si seleziona per sbaglio;
- il manuale idraulico consiglia comunque un'elettrovalvola sullo scarico contro le interruzioni di corrente.

## Formato del log

Ogni riga di log ha una struttura tabellare fissa così da restare **incolonnata**
anche in sessioni lunghe:

```
[HH:mm:ss.fff]  LIVELLO | MITT -> DEST | messaggio
```

- **`MITT`** è chi genera l'informazione, **`DEST`** è chi la riceve/consuma.
- Entrambe le sigle sono **sempre di 4 caratteri**, così la freccia `->` e i due `|` cadono nelle stesse colonne per tutta la giornata.

### Sigle usate

| Sigla  | Cosa rappresenta                                                                                          |
|--------|-----------------------------------------------------------------------------------------------------------|
| `CORE` | Sistema/lifecycle: avvio del servizio, arresto pulito, configurazione non valida, host .NET                |
| `HEXE` | Logica interna del Bridge Hexesoft Fluidra (Manager, routing driver, polling, avvio dispositivi)           |
| `MQTT` | Broker MQTT: connessione, riconnessione, sottoscrizioni, pubblicazioni grezze                             |
| `MODB` | Gateway Modbus TCP e bus RS485 sottostante: transazioni, timeout, riconnessione                            |
| `SCAN` | Bus Scanner: sondaggio degli unit id, riconoscimento del modello dal codice prodotto                       |
| `HOME` | Home Assistant (Auto-Discovery, pulizia dispositivi fantasma)                                              |

### Esempi

```
[15:32:04.001] INFO   | CORE -> HEXE | Hexesoft Bridge Fluidra - Avvio Service
[15:32:04.312] INFO   | HEXE -> MQTT | Tentativo di connessione al Broker...
[15:32:04.500] INFO   | HEXE -> MODB | Connessione al gateway Modbus 192.168.10.42:502...
[15:32:04.612] INFO   | SCAN -> HEXE | Sondo gli indirizzi di fabbrica noti...
[15:32:04.712] INFO   | SCAN -> HEXE | Unit 9: PROELYO INVERBOOST NN (68815)
[15:32:04.713] INFO   | HEXE -> HOME | Pubblicazione entità di 4 dispositivi...
[15:32:15.221] INFO   | HEXE -> MODB | 'Luci Piscina': Luci -> watchdog disattivato all'avvio
[15:32:44.955] INFO   | HEXE -> HOME | Fantasma rimosso: homeassistant/switch/pool_bridge_hexesoft_gpio_26_relay_4/config
```

## Requisiti

- Un **gateway Modbus TCP** (es. **Waveshare RS485-to-PoE-ETH**) raggiungibile in rete sulla **porta 502** (o quella che hai configurato).
- I **parametri seriali** del bus RS485 impostati **uguali** su gateway e su ogni slave Fluidra (default di fabbrica: `9600 8-E-1`).
- Un **broker MQTT** (es. Mosquitto) raggiungibile dal bridge e da Home Assistant.
- **Home Assistant** con l'integrazione **MQTT** attiva (la discovery è abilitata di default).

---

## Guida alla Configurazione

Ogni dispositivo va inserito nella categoria corretta. Tre parametri sono comuni a tutti:

- **`name`**: nome mostrato in Home Assistant.
- **`unit_id`**: indirizzo dello slave sul bus RS485 (1-247). **Deve essere unico** su tutto il bus.
- **`model`** *(opzionale)*: modello hardware, informativo. Se lo lasci vuoto, il `BusScanner` lo compila leggendo il codice prodotto dal dispositivo.
- **`enabled`** *(opzionale, default `true`)*: se `false`, il dispositivo viene ignorato (niente scan, niente polling, niente entità).

### 1. Pompa di Calore (`heat_pumps`)
Pompe **PROELYO INVERBOOST / ELYO SMART B** (indirizzo di fabbrica `9`).

- **`min_temp`** / **`max_temp`**: range del setpoint in HA (default `15` / `40`).
- **`measure_registers`** *(opzionale)*: indirizzi holding delle misure. Se vuoto, `climate` funziona lo stesso ma senza temperatura corrente e senza scrittura del setpoint. Chiavi supportate: `water_in`, `water_out`, `ambient`, `setpoint`.

### 2. Valvola Selettrice (`valves`)
Valvole **SVRAC 2 / SVRAC 3** (indirizzo di fabbrica `11`).

- **`allow_drain`**: se `true`, in Home Assistant compare il pulsante di **SCARICO** (`svuota la piscina`). **Lascia `false`** salvo che tu non abbia un'elettrovalvola di sicurezza sullo scarico e sappia cosa stai facendo (vedi *Sicurezza scarico*).

### 3. GPIO (`gpios`)
Moduli **GP4I-4O** (indirizzo di fabbrica `26`, suggeriti `26-30` per il multi-modulo).

- **`relays`**: lista di 4 elementi. Metti `null` (o ometti) le uscite non cablate.
  - `name`, `type` (`switch` / `light` / `outlet`, default `switch`), `icon` (es. `"mdi:pump"`).
- **`inputs`**: lista di 4 elementi. Metti `null` (o ometti) gli ingressi non cablati.
  - `name`, `device_class` (`problem`, `moisture`, `motion`, …), `expose_counter` (se `true` espone anche il contatore impulsi come sensore diagnostico).

### 4. Luci LumiPlus (`lights`)
Modulo **LumiPlus Modbus** (indirizzo di fabbrica `48`).

- **`sequence_count`**: numero di sequenze predefinite del modulo (default `8`).
- **`colors`**: mappa `Nome colore → codice Modbus`. I codici vanno letti dal manuale LumiPlus Modbus completo; quelli nell'esempio sono segnaposto.

### 5. Dispositivi Generici (`generics`)
Il **jolly** per tutto ciò che non ha un driver dedicato (clorinatori, pompe VS,
regolatori pH/Redox, dosatrici, sensori vari). Si descrivono in JSON leggendo il
manuale Modbus del dispositivo.

- **`prefix`**: prefisso del topic MQTT del dispositivo (es. `"ph_cloro"`, `"chlorinator"`).
- **`mappings`**: lista di registri mappati a entità HA. Per ciascun mapping:
  - **`name`**: nome dell'entità (es. `"pH"`, `"Redox"`, `"Temperatura"`).
  - **`area`**: `holding` / `input` / `coil` / `discrete`.
  - **`address`**: indirizzo del registro.
  - **`kind`**: tipo di entità HA — `sensor`, `binary_sensor`, `switch`, `number`, `select`.
  - **`scale`** *(opzionale)*: fattore di scala (es. `0.01` per un pH memorizzato in centesimi).
  - **`signed`** *(opzionale)*: se `true`, il registro è int16 con segno (per temperature negative).
  - **`unit`** *(opzionale)*: unità di misura HA (es. `"°C"`, `"pH"`, `"mV"`).
  - **`device_class`** *(opzionale)*: `temperature`, `problem`, …
  - **`bit`** *(opzionale)*: per un singolo bit dentro un registro (0-15). `-1` = registro intero.
  - **`writable`** *(opzionale)*: se `true`, l'entità è comandabile (crea un topic `command`).
  - **`min`** / **`max`** / **`step`** *(opzionali)*: solo per `number`.
  - **`diagnostic`** *(opzionale)*: se `true`, finisce nella sezione Diagnostica di HA.

### Esempio di configurazione

```json
{
  "modbus":       { "host": "192.168.10.42", "port": 502, "timeout_ms": 3000 },
  "mqtt":         { "host": "core-mosquitto", "port": 1883, "base_topic": "pool_bridge", "discovery_prefix": "homeassistant" },
  "poll_settings":{ "interval_seconds": 30, "full_bus_scan": false, "scan_from": 1, "scan_to": 60 },

  "heat_pumps": [
    { "name": "Pompa di Calore", "unit_id": 9, "min_temp": 15, "max_temp": 40 }
  ],
  "valves": [
    { "name": "Valvola Selettrice", "unit_id": 11, "allow_drain": false }
  ],
  "gpios": [
    {
      "name": "GPIO Piscina", "unit_id": 26,
      "relays": [
        { "name": "Pompa filtrazione", "type": "switch", "icon": "mdi:pump" },
        { "name": "Pompa idromassaggio", "type": "switch", "icon": "mdi:pump" },
        { "name": "Faretti bordo vasca", "type": "light" },
        null
      ],
      "inputs": [
        { "name": "Flussostato",        "device_class": "problem" },
        { "name": "Livello vasca minimo","device_class": "moisture" },
        { "name": "Contatore backwash", "expose_counter": true },
        null
      ]
    }
  ],
  "lights": [
    { "name": "Luci Piscina", "unit_id": 48, "sequence_count": 8,
      "colors": { "Bianco": 1, "Blu": 2, "Ciano": 3, "Verde": 4, "Magenta": 5, "Rosso": 6 } }
  ]
}
```
