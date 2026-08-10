# Hexesoft Bridge Fluidra

Ponte tra l'impianto piscina **AstralPool / Fluidra** e **Home Assistant**.

Sul lato piscina parla **Modbus RTU** dietro a un gateway **Modbus TCP** (es. Waveshare
RS485-to-Ethernet); sul lato Home Assistant pubblica in **MQTT** con auto-discovery. Ogni
dispositivo Fluidra riconosciuto sul bus diventa una scheda dispositivo in HA con tutte
le sue entità (climate, switch, sensor, select, button), in modo bidirezionale.

---

## Cosa fa in due parole

- **Scopre da solo** cosa hai collegato sul bus RS485 leggendo il blocco identità di
  ogni slave (`HR 0x00-0x0C`) e riconoscendo il modello dal codice prodotto.
- **Configura automaticamente** ogni driver in base al modello letto (indirizzi
  registri, scale, limiti operativi, mappa allarmi): nel file di config l'utente
  scrive solo `name` e `unit_id`.
- **Pubblica su MQTT** tutte le entità di HA con auto-discovery e rimuove quelle
  fantasma quando togli un dispositivo dal file.
- **Ascolta i comandi da HA** e li traduce in scritture Modbus (`FC 0x06` singolo,
  `FC 0x10` multiplo dove serve).
- **Legge ciclicamente** ogni dispositivo (30 s di default), pubblica solo ciò che
  è cambiato dall'ultimo giro, si riconnette da solo a Modbus e MQTT se cade.

---

## Dispositivi supportati

Ogni famiglia è riconosciuta dalla coppia `Manufacturer_hi / Manufacturer_lo`
letta dai registri di identità; il driver corretto viene scelto automaticamente.

| Famiglia AstralPool | mfr hi/lo | Driver | Modelli | Indirizzo di fabbrica |
|---|---|---|---|---|
| **Pompa di calore** PROELYO INVERBOOST NN / PLUS, ELYO SMART B NN | 10072 / 22756 | `HeatPump` | 68815-68823, 68750-68776, 71676-71682 | `9` |
| **Valvola selettrice** SVRAC 2 / SVRAC 3 (± pressostato) | 0 / 5 | `Valve` | 57186, 57187, 70768, 70769, 72433, 72434 | `11` |
| **Modulo GPIO** GP4I-4O (4 relè + 4 ingressi) | 15244 / 11017 | `Gpio` | 62368 | `26` (suggeriti `26-30`) |
| **Modulo luci** LumiPlus Modbus | 0 / 14 | `Light` | 57434, 57435 | `48` (`0x30`) |
| **Centralina chimica** Energy Connect / Tecno LC2 | 0 / 178 | `EnergyConnect` | tutte le varianti KA/AP | (variabile) |
| **Qualunque altro dispositivo Modbus** | — | `Generic` | mappings da manuale | (utente) |

Non c'è un elenco chiuso: se colleghi un modello nuovo della stessa famiglia (es. un
nuovo prodotto della serie PROELYO), il bridge lo riconosce dalla `mfr` e usa il
profilo giusto — al massimo il `product_code_lo` finisce con un nome generico tipo
"Pompa di calore AstralPool (codice X)" ma tutto il resto funziona.

---

## Come funziona (dietro le quinte)

### Connessione e serializzazione
- Una sola connessione TCP verso il gateway Modbus (porta `502`).
- Tutte le transazioni sul bus RS485 sono serializzate con un semaforo interno: sul
  bus c'è un solo master (noi) e sovrapporre due richieste = perdere entrambe.
- Parametri seriali del bus (default `9600 8-E-1`) devono coincidere fra gateway e
  ogni slave.

### BusScanner (avvio)
- Sonda per primi gli **indirizzi di fabbrica documentati** (pompa `9`, valvola `11`,
  GPIO `26`, LumiPlus `48`, ecc.): pochi secondi, trova tutto in impianti "puliti".
- Solo se richiesto (`full_bus_scan: true`) scandisce l'intervallo completo `scan_from
  .. scan_to`, con `SCAN_DELAY_MS = 30 ms` per unit id assente.
- Per ogni slave che risponde: legge `HR 0x00-0x0C`, verifica che l'auto-diagnosi
  (`HR 0x00 == unit_id` richiesto) sia coerente, riconosce famiglia + modello.
- **Chiama `OnIdentityDetected(identity)`** sul driver, che applica il profilo giusto
  in base al codice prodotto (es. `ApplyProfileProelyoInverboostNn()`).

### Polling
- Loop ogni `interval_seconds` (default `30 s`).
- Ogni driver dichiara un `GetPollPlan()` con blocchi Modbus contigui da leggere
  (`RegisterArea` = Holding / Input / Coil / Discrete).
- I blocchi **`Optional`** (es. misure sconosciute, registri non tutti mappati) non
  fanno cadere il device in caso di `Illegal Data Address`: `Values` resta `null` e
  il polling continua.
- Il driver traduce i registri letti in `PublishInfo` (topic + payload MQTT).
- **Dedup automatico**: `AddIfChanged` pubblica solo se il payload è diverso
  dall'ultimo pubblicato per quello specifico suffisso.
- Il PollingWorker distanzia i device sul bus con un piccolo delay + retry per
  stabilità del bus RS485 half-duplex a 9600 baud.

### Comandi HA → Modbus
- Sottoscrizione al topic `pool_bridge/#` e al `homeassistant/#` (per pulizia
  fantasmi).
- Ogni driver espone `GetModbusCommand(topicSuffix, payload)` che ritorna la
  scrittura Modbus da fare (concatenabile con `.Then(...)` per operazioni a più passi
  come cambio sequenza LumiPlus o scarico valvola).
- **Publish ottimistico**: subito dopo la scrittura, il bridge pubblica lo stato
  atteso — così l'interruttore in HA non si "annulla" visivamente aspettando il
  polling successivo.

### Pulizia entità e state topic fantasma
- All'avvio, dopo la pubblicazione delle entità configurate, il bridge scansiona
  `homeassistant/#` e `pool_bridge/#` e rimuove entità di configurazione + state
  topic che non appartengono più a nessun device configurato.
- Se togli un dispositivo dal file, HA non resta con entità morte "sconosciute".

### Auto-configurazione dei driver dall'identità
- La classe base `FluidraDevice` espone un hook virtuale `OnIdentityDetected
  (DeviceIdentity)` chiamato dal `BusScanner` subito dopo il probe.
- I driver che ne hanno bisogno (per ora `HeatPump`) selezionano il profilo giusto
  guardando `Identity.ManufacturerHi/Lo` e `Identity.ProductCodeLo`.
- Vantaggio: se domani sostituisci una PROELYO NN con una ELYO SMART B, non tocchi
  nulla nella config — il bridge riconosce il nuovo modello e applica la sua mappa.

---

## Dispositivi in dettaglio

### 1. Pompa di calore — `heat_pumps`

Driver `HeatPump.cs`. Testato su **PROELYO INVERBOOST NN 68823** (SW 1030). La stessa
mappa registri copre tutta la serie PROELYO NN/PLUS e la ELYO SMART B NN.

**Entità Home Assistant esposte**

| Entità | Tipo HA | Cosa fa |
|---|---|---|
| Climate principale | `climate` | Modes `off / heat / cool / auto` (tradotti da HA in "Spento / Riscaldamento / Raffreddamento / Automatico"), setpoint 15-40 °C con step 0.5, current temperature = Water Inlet |
| Preset inverter | preset di climate | `Silenzioso / Intelligente / Potente` (identici al portale Fluidra Connect) |
| Action topic | action di climate | `off / heating / cooling / idle` — stato REALE (letto dal bit Compressor dello Status), non richiesta |
| Acqua ingresso | `sensor` °C | IR 0x07 — temperatura acqua dalla piscina |
| Acqua uscita | `sensor` °C | IR 0x09 — temperatura acqua verso piscina |
| Temperatura ambiente | `sensor` °C | HR 0x34 |
| Gas exhaust | `sensor` °C | IR 0x1C — mandata compressore |
| Gas return | `sensor` °C | IR 0x1B — ritorno dal condensatore |
| EVI step | `sensor` | IR 0x0C — apertura valvola espansione (18-94) |
| Contatore avvii compressore | `sensor` totalizzatore | HR 0x39 |
| Compressore | `binary_sensor` (diagnostica) | Bit 6 Status |
| Sbrinamento | `binary_sensor` (diagnostica) | Bit 5 Status |
| Modo pompa acqua | `sensor` (diagnostica) | Bit 3 Status (`Confort` / `Filtration`) |
| Guasto | `binary_sensor` (problem) | OR degli allarmi critici (16 codici PP/EE del manuale) |
| Descrizione errore | `sensor` testo | Lista degli allarmi attivi in italiano |
| Reset allarmi | `button` (config) | Scrive 0 in `HR 0x20` (allarmi latched) |

**Comportamenti particolari** (documentati nel commento di classe `HeatPump.cs`)

- Il bit RUN del Control Word viene abbassato **autonomamente dalla pompa** quando
  entra in standby (temperatura raggiunta). Se lo interpretassimo come "Off", HA
  ballerebbe fra Heat e Spento: perciò il MODE deriva **dai soli bit 1-2 (HP Mode)**
  del CW, ignorando RUN.
- L'OFF esplicito da HA setta un flag interno `_userExplicitOff = true`: HA vede
  `off` finché non viene chiesta esplicitamente una MODE (`heat`/`cool`/`auto`).
- **Al riavvio del bridge** se la pompa risulta con RUN=0 → il flag viene settato a
  `true` (HA vede OFF invece del MODE fantasma). Se poi la pompa viene riaccesa
  esternamente (dal display fisico) → il flag viene resettato al primo polling che
  vede RUN=1 → HA torna a mostrare il MODE.
- La PROELYO può sovrascrivere il preset scelto (Silent/Powerful) tornando a Smart
  per auto-regolazione: non è un bug del bridge, è la logica interna della PdC.

**Sonda esplorativa opzionale**

Il driver ha un flag `enable_debug_probe` (default `false`). Se attivato:
- Aggiunge ~12 blocchi Optional al poll plan che leggono i registri candidati non
  ancora identificati (IR `0x02-0x06`, `0x08-0x0B`, `0x0D-0x34`, HR `0x22-0x38`).
- In log `debug` stampa una sezione `[SONDA]` con i raw hex di ogni registro (`---`
  se il registro risponde Illegal Data Address).
- Costo: ~500-800 ms extra ogni giro. Zero costo se disattivato.

Serve per capire in un log di diagnostica se qualche registro sconosciuto cambia
durante un'operazione (defrost, cambio preset, ecc.), senza dover ricompilare.

**Configurazione utente**: solo `name` + `unit_id` (+ opz. `enabled`). Tutti gli
indirizzi Modbus, la scala setpoint (decimi °C), i limiti temperatura sono hardcoded
nel driver in base al modello riconosciuto.

---

### 2. Valvola selettrice — `valves`

Driver `Valve.cs`. Testato sulla logica della **SVRAC 3 2"** (70769).

**Entità Home Assistant esposte**

| Entità | Tipo HA | Cosa fa |
|---|---|---|
| Posizione | `select` | `Filtrazione / Chiusa / Ricircolo` — SCARICO NON è qui per sicurezza |
| Stato attuale | `sensor` | Posizione REALE letta (7 valori: Closed, Filtration, Drain, Circulation, Backwash, Rinse, InTransit) |
| Ciclo di lavaggio | `button` | Sequenza automatica backwash + risciacquo |
| Scarico | `button` (solo se `allow_drain: true`) | Sequenza a due passi con conferma entro 10 s |
| Guasto | `binary_sensor` (problem) | OR dei 11 allarmi "hard" (micro, sovraccarico motore, ecc.) |
| Descrizione errore | `sensor` testo | Lista dei 14 codici allarme in italiano |
| Reset allarmi | `button` (config) | Scrive 0 in `HR 0x20` |

**Sicurezza scarico**

La posizione `Scarico` manda l'acqua della piscina **in fogna**. Per questo:
1. **Il costruttore** ha messo una sicurezza a due passi: "Request DRAIN" apre una
   finestra di 10 s, "Confirm DRAIN" dentro quella finestra esegue davvero. Il
   bridge rispetta questa sequenza.
2. **Il bridge**: `allow_drain: false` di default → **nessun pulsante Scarico**
   compare in HA. Chi lo abilita lo fa consapevolmente.
3. **Il pulsante**, quando abilitato, è **separato** dal dropdown delle posizioni —
   non lo puoi selezionare per sbaglio dalla lista.
4. **Il manuale idraulico** consiglia un'elettrovalvola meccanica di sicurezza sullo
   scarico contro le interruzioni di corrente (specie se l'impianto è sotto il
   livello della piscina).

**Fallback Write Multiple Registers**: alcuni firmware SVRAC rifiutano `FC 0x06`
(Write Single) e vogliono `FC 0x10` (Write Multiple). Il driver ritenta
automaticamente con FC 0x10 se la scrittura singola fallisce.

**Configurazione utente**: `name` + `unit_id` + `allow_drain` (+ opz. `enabled`).

---

### 3. Modulo GPIO — `gpios`

Driver `Gpio.cs`. Testato su **GP4I-4O SW 119**. Mappa registri completa dal manuale
ufficiale AstralPool.

**Struttura fissa**: il modulo ha SEMPRE 4 relè + 4 ingressi. Nel file di config i
loro slot sono FISSI (`relay_1` .. `relay_4`, `input_1` .. `input_4`) — non è una
lista dinamica. Vantaggio: la corrispondenza indice → hardware è garantita, e
nella UI di HA non c'è più il bug "modifico un elemento della lista → ne aggiunge
uno nuovo".

Ogni slot ha:
- `enabled`: se `false`, quello slot non genera entità in HA (ma il driver continua
  a leggere il registro — è lo stesso blocco).
- `name`: nome mostrato in HA. Se vuoto, "Relè N" / "Ingresso N".
- `type`: **preset piscina** — un dropdown italiano da cui il driver deriva
  automaticamente entity kind + device_class + icona.

**Preset relè disponibili**

| `type` | Entity HA | Icona |
|---|---|---|
| Interruttore | switch | (default) |
| Pompa | switch | 💧 mdi:pump |
| Elettrovalvola | switch | 🔧 mdi:valve |
| Ventilatore | switch | 🌀 mdi:fan |
| Riscaldatore | switch | 🔥 mdi:radiator |
| Presa elettrica | switch | 🔌 mdi:power-socket-it |
| Luce | light | 💡 mdi:lightbulb |
| Faretti LED | light | ✨ mdi:led-strip-variant |

**Preset ingressi disponibili**

| `type` | binary_sensor | counter | device_class |
|---|---|---|---|
| Contatto generico | ✅ | — | — |
| Flussostato | ✅ | — | problem |
| Pressostato | ✅ | — | problem |
| Livello acqua | ✅ | — | moisture |
| Movimento | ✅ | — | motion |
| Porta / finestra | ✅ | — | door |
| Fumo | ✅ | — | smoke |
| Contatore impulsi | — | ✅ | (solo counter, no ON/OFF) |
| Contatto + contatore | ✅ | ✅ | problem |

**Contatori impulsi**: layout dipendente dal firmware:
- **SW < 120**: contatori 16-bit su `IR 0x03..0x06` (uno per registro).
- **SW ≥ 120**: contatori 32-bit su `IR 0x03..0x0A` (due registri per contatore,
  hi + lo). Il driver adatta il piano di lettura leggendo `Identity.SwVersion`.

**Polarità dei relè**: `1 = relè CHIUSO/attivo (ON)`. **Lo decide l'hardware, non il
manuale**: le versioni inglese e italiana del manuale si contraddicono fra loro sulla
polarità (errore di traduzione) e, preso alla lettera, il manuale suggerirebbe persino
`1 = aperto`. Ma sul GP4I-4O reale il comando ON → il driver scrive bit=1 → il motore
si accende. Costante `INVERT_RELAY_LOGIC = false`. Su un modulo diverso, ri-verificare
a banco con un carico innocuo.

---

#### Cosa vedi in Home Assistant — guida per l'utente

Il bridge **non usa mai il portale Fluidra**: tutto il modulo si configura da qui. La
scheda del GPIO in HA è divisa in tre riquadri.

##### 🔵 Controlli — l'uso di tutti i giorni
Un interruttore per ogni relè che hai attivato (es. "Motore 1", "Motore 2"). Lo
accendi/spegni e il relè si chiude/apre subito. **È l'unica parte che userai
normalmente.**

##### ⚙️ Configurazione — impostazioni di sicurezza (si toccano una volta e si dimenticano)

- **Watchdog comunicazione** (secondi): timer di sicurezza. Se il bridge/HA smette di
  parlare col modulo per più di questo tempo, i relè vengono messi in uno stato "di
  sicurezza" predefinito.
  - **`0` = disattivato → CONSIGLIATO per la piscina**: se il bridge cade, i relè
    RESTANO come sono (la pompa di filtrazione continua a girare). È il default.
  - Un valore > 0 (es. `60`) ha senso solo per carichi che è pericoloso lasciare accesi
    senza controllo. Nota: sotto i 30s il modulo lo tratta comunque come 30s.
- **Azione al watchdog**: se hai attivato il watchdog, decide DOVE vanno i relè quando
  scatta:
  - *Applica 'Relè al watchdog'* → usano gli switch **"— al watchdog"** qui sotto;
  - *Applica 'Relè all'accensione'* → usano gli switch **"— all'accensione"**.
- **… — al watchdog** (uno per relè): come deve stare quel relè SE scatta il watchdog
  (interruttore ON = chiuso). Serve solo con il watchdog attivo.
- **… — all'accensione** (uno per relè): come deve stare quel relè appena il modulo
  prende corrente (ON = chiuso). Utile p.es. per far partire la filtrazione da sola
  all'accensione, anche senza HA.
- **Filtro anti-rimbalzo ingressi**: serve **solo** se usi gli ingressi come contatori
  di impulsi. I contatti meccanici "rimbalzano" e rischiano di contare due volte; il
  filtro (1/10/100/500 ms) ignora i rimbalzi. Lascia **Off** se non usi contatori.
- **Reset allarmi**: pulsante. Azzera l'allarme del watchdog una volta capito perché è
  scattato.

##### 📊 Diagnostica — sola lettura, per controllare la salute del modulo
- **Accensioni**: quante volte il modulo ha preso corrente (nella tua foto: `240`).
- **Scatti watchdog**: quante volte è scattato il watchdog (`0` = mai, ottimo). Se
  cresce, il bridge perde contatto col modulo troppo spesso → controlla cavo/bus RS-485.
- **Stato modulo**: stato interno — `Start` / `Richiesta comando` / `Watchdog scattato`.
- **Watchdog scattato**: `OK` finché tutto va bene; segnala se l'allarme è memorizzato.

> **In breve, per una piscina normale**: lascia **Watchdog = 0** e **Filtro = Off**, e
> usa solo i due interruttori dei relè nel riquadro *Controlli*. Tutto il resto è per chi
> vuole comportamenti di failsafe particolari.

---

**Watchdog Modbus (dettaglio tecnico)**: `HR 0x10` Watchdog_time (sec, 0=off), `HR 0x11`
Watchdog_config (byte alto: `0` = stato Watchdog → relè a `HR 0x14`; `≠0` = stato Start →
relè a `HR 0x15`), `HR 0x14` wdt_relay_state, `HR 0x15` start_relay_state. Tutti esposti
in HA come entità *config*. Il bridge **non forza più** `HR 0x10 = 0` all'avvio (ora lo
controlli tu da HA); l'unica scrittura di avvio è `HR 0x20 = 0` (reset allarmi latched dal
boot precedente).

**Registri config/diagnostica esposti** (3 blocchi Modbus *Optional* aggiuntivi per giro):
`HR 0x10-0x12`, `HR 0x14-0x15`, `HR 0x30` (power_counter) / `0x31` (WDT_counter), + `IR 0x00`
byte alto (stato macchina). Se un firmware non li implementasse, il device NON va offline
(blocco saltato).

**Configurazione utente** (nel form/file dell'add-on): solo `name` + `unit_id` + 8 slot
(`relay_1..4`, `input_1..4`) con `enabled`, `name`, `type`. **Nessun** indirizzo Modbus,
device_class inglese o `mdi:*`. I parametri di watchdog/filtro/failsafe **non** stanno nel
form di setup: si regolano a runtime dalle entità HA descritte sopra.

---

### 4. Modulo luci LumiPlus — `lights`

Driver `Light.cs`. Mappa Modbus completa dal manuale ufficiale AstralPool
(`Lumiplus.pdf` cap. 9).

**Entità Home Assistant esposte**

| Entità | Tipo HA | Cosa fa |
|---|---|---|
| Luci Piscina | `switch` | On/Off (comando bit 0 o bit 1 di `HR 0x21`) |
| Colore | `select` | 12 colori: `Rosso, Verde, Blu, Giallo, Ciano, Magenta, Viola pallido, Blu cielo, Arancione, Rosa, Turchese, Bianco` — **identici al portale Fluidra Connect** |
| Sequenza | `select` | 8 sequenze coreografiche con nomi ufficiali: `Eleven, Abril, Paradise, Tropical, Candy, Iris, Caroline, Estival` |
| Velocità | `number` slider 1-8 | Velocità di transizione della sequenza (1=lento, 8=veloce) |
| Sospensione | `select` | Timer spegnimento: `off / 5 min / 15 min / 30 min / 60 min / 90 min / 120 min / 240 min / 480 min` |
| Modalità | `sensor` (diagnostica) | Letto dallo Status: `Idle / Watchdog / Colore fisso / Sequenza in corso` — utile per conditional card in HA |
| Watchdog scattato | `binary_sensor` (problem) | Bit 15 di `IR 0x01` |
| Reset allarmi (sblocca) | `button` (config) | Scrive 0 in `HR 0x20` — **CRITICO**: finché il watchdog è latched il LumiPlus rifiuta ogni comando |

**Cambio colore / sequenza è a due passi** (manuale §8.1.6-8.1.8):
1. Scrivere il valore in `HR 0x25` (colore) / `HR 0x26` (sequenza) / `HR 0x27` (velocità)
2. Scrivere il bit di "richiesta aggiornamento" in `HR 0x21` (bit 3 per colore, bit
   4 per sequenza + velocità)

Il driver concatena le due scritture con `ModbusWrite.Then()`.

**Watchdog disattivato all'avvio**: il bridge scrive `HR 0x10 ← 0` non appena vede
il modulo online — se lasciato attivo, la prima volta che il bridge tace le luci
vanno al colore scritto in `HR 0x14`.

**Unidirezionalità** (manuale §7.1): se qualcuno tocca il telecomando fisico delle
luci, il modulo Modbus **non lo sa**. Lo Status `IR 0x00` rispecchia solo comandi
via Modbus, non lo stato reale della lampada. Per questo l'entità principale è
`optimistic`.

**Configurazione utente**: solo `name` + `unit_id` (+ opz. `enabled`).

---

### 5. Centralina cloro/pH — `chlorinators`

Driver `EnergyConnect.cs` per **AstralPool Energy Connect / Tecno LC2**. Mappa
Modbus dal manuale ufficiale.

**Grandezze misurate (sensori)**

| Entità | Tipo HA | Cosa fa |
|---|---|---|
| pH | `sensor` | valore attuale |
| ORP | `sensor` mV | redox |
| Cloro libero | `sensor` ppm | cloro effettivo in acqua |
| Temperatura | `sensor` °C | sonda temperatura acqua |
| Sale | `sensor` g/l | salinità (per impianti a elettrolisi) |
| Produzione istantanea | `sensor` % | % di elettrolisi attualmente in corso |
| Cloro prodotto ora / oggi / totale | `sensor` g | contatori grammi cloro |
| Corrente elettrodi | `sensor` A (diagnostica) | consumo cella elettrolitica |
| Tensione elettrodi | `sensor` V (diagnostica) | tensione cella elettrolitica |

**Setpoint e comandi**

| Entità | Tipo HA | Cosa fa |
|---|---|---|
| Setpoint pH | `number` | target pH desiderato |
| Setpoint ORP | `number` | target ORP |
| Setpoint produzione cloro | `number` % | % elettrolisi target |
| Modalità pH | `select` | acido / basico |
| Calibra pH 7 / pH 4 / ORP 470 mV | `button` (config) | calibrazione sonde |
| Boost cloro | `button` | produzione massima temporanea |
| Reset ore parziali elettrolisi | `button` (config) | azzera contatore ore parziali |
| Riavvia centralina | `button` (config) | reset software del modulo |
| Reset allarmi pH / ORP / temp / sale / flusso / elettrolisi | `button` (config) | reset per famiglia di allarmi |

**Diagnostica allarmi**

`Allarme pH / ORP / sale / temperatura / flusso` + `Check cell (manutenzione
elettrodi)` + `Problema generico` come `binary_sensor`. Anche il risultato
dell'ultima calibrazione è un `sensor` testo diagnostico.

**Configurazione utente**: solo `name` + `unit_id` (+ opz. `enabled`).

**Nota**: il TLC2 non segue il framework "standard" Fluidra al 100%
(`COM_Setup` di default 1152 invece di 0, `MODEL_Serie` su 3 registri hi/mi/lo,
`0x06` è `ID_Technologies_implemented` invece di "Reserved") — la sua identità è
gestita a parte nel `ProductCatalog`.

---

### 6. Driver Generico — `generics`

Il **jolly** per qualsiasi dispositivo Modbus non catalogato (pompe di ricircolo
VS, regolatori pH/Redox esterni, dosatrici, sensori vari). La mappa registri si
descrive direttamente nel file di config leggendo il manuale Modbus del prodotto —
non serve compilare codice.

Ogni mapping può produrre:
- **sensor** (letture)
- **binary_sensor** (contatti / bit)
- **switch** (scritture ON/OFF su un bit o un intero)
- **number** (setpoint numerici con `min`/`max`/`step`)
- **select** (dropdown)
- **light** (interruttore luce)

Ogni mapping supporta: `area` (holding/input/coil/discrete), `address`, `bit`
(0-15 o `-1` per registro intero), `scale`, `signed` (per int16), `unit`,
`device_class`, `icon`, `writable`, `min`/`max`/`step`, `diagnostic`.

**Configurazione utente**: `name` + `unit_id` + `prefix` (per il topic MQTT) +
`mappings[]` (con l'intera mappa registri che ti interessa esporre).

---

## Configurazione — UI minimale

**Principio**: la UI di Home Assistant chiede all'utente solo quello che l'utente
può realmente sapere. Tutti gli indirizzi Modbus interni, le scale numeriche, i
limiti operativi, le device_class di HA sono **hardcoded nel driver** e vengono
selezionati automaticamente dopo il SCAN.

Per la maggior parte dei device servono **solo tre campi**:
- **`name`**: nome mostrato in HA
- **`unit_id`**: indirizzo Modbus (1-247, unico sul bus)
- **`enabled`** *(opz., default `true`)*: se `false`, il device è ignorato

Le eccezioni sono i device che hanno **cablaggio fisico personale** o **scelte di
sicurezza**:
- **GPIO**: 8 slot fissi (4 relè + 4 input) con preset in italiano
- **Valvola**: `allow_drain` (bool) per abilitare il pulsante Scarico
- **Generic**: `prefix` + `mappings[]`

### Esempio `appsettings.json`

```json
{
  "modbus":       { "host": "192.168.10.42", "port": 502, "timeout_ms": 3000 },
  "mqtt":         { "host": "core-mosquitto", "port": 1883, "base_topic": "pool_bridge", "discovery_prefix": "homeassistant" },
  "poll_settings":{ "interval_seconds": 30, "full_bus_scan": false, "scan_from": 1, "scan_to": 60 },
  "system_settings":{ "log_level": "info" },

  "heat_pumps": { "devices": [
    { "name": "Pompa di Calore", "unit_id": 9 }
  ]},

  "valves": { "devices": [
    { "name": "Valvola Selettrice", "unit_id": 11, "allow_drain": false }
  ]},

  "gpios": { "devices": [
    {
      "name": "GPIO Piscina", "unit_id": 26,
      "relay_1": { "enabled": true, "name": "Motore 1", "type": "Pompa" },
      "relay_2": { "enabled": true, "name": "Motore 2", "type": "Pompa" },
      "relay_3": { "enabled": false },
      "relay_4": { "enabled": false },
      "input_1": { "enabled": true, "name": "Flussostato", "type": "Flussostato" },
      "input_2": { "enabled": false },
      "input_3": { "enabled": false },
      "input_4": { "enabled": false }
    }
  ]},

  "lights": { "devices": [
    { "name": "Luci Piscina", "unit_id": 48 }
  ]},

  "chlorinators": { "devices": [
    { "name": "Energy Connect", "unit_id": 2 }
  ]}
}
```

---

## Log

Formato tabellare fisso per rimanere **incolonnato** anche in sessioni lunghe:

```
[HH:mm:ss.fff]  LIVELLO | MITT -> DEST | messaggio
```

- Sigle `MITT` e `DEST` sempre di 4 caratteri, così frecce e separatori cadono
  nella stessa colonna a ogni riga.
- Livelli: `INFO`, `DEBUG`, `WARN`, `ERROR`.
- Log salvato anche su file (`log-fluidra-YYYYMMDD-HHMMSS.txt` nella directory di
  esecuzione).

**Sigle**

| Sigla | Cosa rappresenta |
|---|---|
| `CORE` | Sistema/lifecycle: avvio del servizio, arresto pulito, configurazione |
| `HEXE` | Logica interna del Bridge Hexesoft Fluidra |
| `MQTT` | Broker MQTT: connessione, sottoscrizioni, publish grezzi |
| `MODB` | Gateway Modbus TCP + bus RS485: transazioni, timeout, riconnessione |
| `SCAN` | Bus Scanner: sondaggio unit id, riconoscimento modello |
| `HOME` | Home Assistant (Auto-Discovery, pulizia fantasmi) |
| `DBG`  | Diagnostica dettagliata per il driver (bit-per-bit di un CW, misure raw) |

**Esempio**

```
[15:32:04.001] INFO   | CORE -> HEXE | Hexesoft Bridge Fluidra - Avvio Service
[15:32:04.312] INFO   | HEXE -> MQTT | Tentativo di connessione al Broker...
[15:32:04.712] INFO   | SCAN -> HEXE | Unit 9: PROELYO INVERBOOST NN (68823)
[15:32:04.712] INFO   | SCAN -> HEXE |          prod 1/3287 | mfr 10072/22756 | HW 12291 | SW 1030 | 9600 8-E-1
[15:32:15.221] INFO   | HEXE -> MODB | 'Luci Piscina': Luci -> watchdog disattivato all'avvio
[15:32:44.955] INFO   | HEXE -> HOME | Fantasma rimosso: homeassistant/switch/pool_bridge_hexesoft_gpio_26_relay_4/config
```

---

## Requisiti

- **Gateway Modbus TCP** (es. **Waveshare RS485-to-ETH**) raggiungibile sulla porta
  `502` (o quella che hai configurato).
- **Parametri seriali** del bus RS485 uguali su gateway e su ogni slave Fluidra
  (default `9600 8-E-1`).
- **Broker MQTT** (es. Mosquitto, o l'add-on ufficiale HA) raggiungibile dal bridge
  e da Home Assistant.
- **Home Assistant** con integrazione **MQTT** attiva (discovery abilitata di
  default sul prefisso `homeassistant/`).
- **.NET 9** runtime (se non usi il container / add-on HA).

---

## Diagnostica e troubleshooting

- **Log level `debug`**: mostra bit-per-bit i Control Word / Status dei device
  complessi (pompa di calore, luci), le misure raw prima della scalatura, i
  suffissi MQTT pubblicati per ogni giro.
- **Sonda esplorativa pompa di calore** (`enable_debug_probe: true`): stampa i
  raw hex dei ~60 registri candidati non ancora identificati.
- **`full_bus_scan: true` + pulsante Bridge → Scansione**: ripete lo scan
  dell'intervallo `scan_from..scan_to` (lento — usare quando si aggiunge un
  dispositivo con indirizzo custom).
- **Watchdog latched**: se un modulo (LumiPlus soprattutto) rifiuta i comandi
  Modbus, quasi sempre è il watchdog scattato. Il bottone `Reset allarmi (sblocca)`
  in HA lo azzera.
- **Bus saturato o timeout su comandi**: verificare che `interval_seconds` non sia
  troppo aggressivo (30 s è un buon punto di partenza), che il `poll plan` non
  abbia bloccati optional che generano continui retry, che non ci sia un altro
  master sul bus.
