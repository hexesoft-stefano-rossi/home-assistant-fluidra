# Hexesoft Bridge Fluidra

## Descrizione del Sistema
**Hexesoft Bridge Fluidra** è un'applicazione per interfacciarsi con la gamma
domotica **AstralPool / Fluidra** (bus **Modbus RTU** dietro gateway Modbus TCP,
tipicamente **Waveshare RS485-to-ETH**). Il software funge da ponte (bridge)
tra il gateway fisico e **Home Assistant**, utilizzando il protocollo **MQTT**
per la comunicazione.

### Funzionalità Principali
* **Auto-Discovery**: all'avvio pubblica in Home Assistant tutte le entità dei dispositivi configurati e rimuove i "fantasmi" delle entità non più presenti.
* **Bus Scanner**: scopre da solo cosa c'è sul bus RS485 sfruttando la proprietà del framework Fluidra per cui l'holding register `0x00` di ogni dispositivo contiene il proprio indirizzo (autodiagnosi).
* **Pompa di Calore** (PROELYO INVERBOOST / ELYO SMART B): entità `climate` con modes off/heat/cool/auto, preset silent/smart/powerful, setpoint e temperatura corrente (se gli indirizzi delle misure sono forniti), diagnostica dei 32 allarmi PP**/EE**.
* **Valvola Selettrice SVRAC**: posizione (Filtrazione/Chiusa/Ricircolo), pulsante di lavaggio+risciacquo, allarmi. Il pulsante di SCARICO è disabilitato di default per sicurezza.
* **Modulo GPIO GP4I-4O**: 4 relè liberamente configurabili (switch/light) e 4 ingressi digitali (con `device_class` HA e opzionale contatore impulsi).
* **Luci LumiPlus**: on/off, colore predefinito (select), sequenza e velocità (number). L'entità è marcata `optimistic` perché il modulo Modbus è unidirezionale (il telecomando/pulsanti a muro non aggiornano lo stato letto via Modbus).
* **Driver Generico**: qualunque dispositivo Modbus può essere descritto direttamente nel file di configurazione tramite `mappings` (area/indirizzo/scala/kind), senza scrivere codice.
* **Watchdog**: non attivato di default (un bridge che si ferma non deve far scattare i relè della piscina). Le luci hanno una scrittura di avvio che lo azzera esplicitamente.

## System Description
**Hexesoft Bridge Fluidra** is an application designed to interface with the
**AstralPool / Fluidra** pool automation range (Modbus RTU behind a Modbus TCP
gateway, typically **Waveshare RS485-to-ETH**). The software acts as a bridge
between the physical gateway and **Home Assistant**, using the **MQTT**
protocol for communication.

### Main Features
* **Auto-Discovery**: on startup, publishes all Home Assistant entities of configured devices and removes ghost entities that are no longer present.
* **Bus Scanner**: auto-discovers what is on the RS485 bus by exploiting the Fluidra framework property that holding register `0x00` of every device contains its own address (self-diagnostic).
* **Heat Pump** (PROELYO INVERBOOST / ELYO SMART B): HA `climate` entity with off/heat/cool/auto modes, silent/smart/powerful presets, setpoint and current water temperature (when measure register addresses are provided), diagnostic of 32 PP**/EE** alarms.
* **Selector Valve SVRAC**: position (Filtration/Closed/Recirculation), backwash+rinse button, alarms. The DRAIN button is disabled by default for safety.
* **GPIO Module GP4I-4O**: 4 configurable relays (switch/light) and 4 digital inputs (with HA `device_class` and optional pulse counter).
* **LumiPlus Lights**: on/off, preset color (select), sequence and speed (number). The entity is marked `optimistic` because the Modbus module is one-way (remote/wall buttons do not update the Modbus-read state).
* **Generic Driver**: any Modbus device can be described directly in the configuration file via `mappings` (area/address/scale/kind), with no code changes.
* **Watchdog**: not enabled by default (a bridge that stops should not make the pool relays trip). LumiPlus lights have a startup write that explicitly clears it.
