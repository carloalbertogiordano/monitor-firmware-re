# Prompt di handoff — Reverse engineering firmware monitor "D1"

Incolla questo prompt per intero al modello AI che continuerà il lavoro.

---

## Obiettivo

Trovare e correggere un bug nel firmware di un monitor generico/OEM no-name (nome interno "D1"). Il bug: il monitor si accende a ~200% di luminosità e va abbassato manualmente a 100%. L'ipotesi di lavoro è che manchi un **clamp/limite massimo** — se arriva una richiesta di luminosità sopra il 100%, il firmware dovrebbe limitarla a 100% ma non lo fa.

Non stiamo cercando di ricompilare il firmware da zero (non abbiamo compilatore/SDK originali) — l'obiettivo finale è una **patch binaria minima**: trovare i pochi byte responsabili e modificarli direttamente nel file `.bin`, lasciando il resto intatto.

## Hardware e contesto

- Monitor generico OEM, connesso a un laptop TUXEDO InfinityBook S Gen8 via USB-C
- Chip video scaler: **RTD2525BE** (Realtek, serie LRH) — QFN, pilota pannello LVDS
- Chip separato USB-C PD: **CPW3220** (Chipsea, ARM Cortex-M0) — gestisce un bug di carica USB-C diverso e non ancora indagato, NON è rilevante per il bug di luminosità
- Flash SPI esterna dumpata: **ZB25VQ80** (Zbit Semiconductor, 1MB, chip 3V) — contiene il firmware oggetto di questa analisi
- Il dump (`firmware_dump_1.bin` / `firmware_dump_2.bin` in questa cartella, identici, verificati via SHA256) è stato letto con CH341A + clip SOP8 + `flashrom -p ch341a_spi -r`

## Struttura del firmware — IMPORTANTE

Il processore che esegue questo codice è un **8051** (confermato: tabella vettori di interrupt standard trovata a CODE:0000, big-endian). L'8051 indirizza solo 64KB di codice alla volta, ma il file è 1MB → **usa bank-switching**. Verificato che il file contiene **16 banchi da 64KB**, di cui solo alcuni hanno contenuto reale (il resto è 0xFF/vuoto):

| Banco | Offset nel file | Contenuto |
|---|---|---|
| 0 | 0x000000 | SÌ — già analizzato in Ghidra (vedi sotto) |
| 1 | 0x010000 | SÌ |
| 2 | 0x020000 | SÌ |
| 3 | 0x030000 | SÌ |
| 4 | 0x040000 | SÌ |
| 5 | 0x050000 | SÌ (parziale, 26%) |
| 6 | 0x060000 | NO (vuoto) |
| 7 | 0x070000 | SÌ (parziale, 17%) |
| 8-12 | — | NO (vuoti) |
| 13 | 0x0d0000 | SÌ (parziale, 9%) |
| 14-15 | — | NO (vuoti) |

**Importazione in Ghidra**: dato che gli overlay memory block hanno dato problemi nella UI di Ghidra 12.1.2 usata finora, la strada scelta è importare **ogni banco come programma Ghidra separato** (stesso file, offset diverso), non come overlay nello stesso programma:

| Nome programma | Language | Base Address | File Offset | Length |
|---|---|---|---|---|
| BANK0 (già fatto) | 8051:BE:16:default | 0x0000 | 0x00000 | 0x10000 |
| BANK1 | 8051:BE:16:default | 0x0000 | 0x10000 | 0x10000 |
| BANK2 | 8051:BE:16:default | 0x0000 | 0x20000 | 0x10000 |
| BANK3 | 8051:BE:16:default | 0x0000 | 0x30000 | 0x10000 |
| BANK4 | 8051:BE:16:default | 0x0000 | 0x40000 | 0x10000 |
| BANK5 | 8051:BE:16:default | 0x0000 | 0x50000 | 0x10000 |
| BANK7 | 8051:BE:16:default | 0x0000 | 0x70000 | 0x10000 |
| BANK13 | 8051:BE:16:default | 0x0000 | 0xd0000 | 0x10000 |

**Nota importante**: dopo l'import con "Raw Binary", l'auto-analisi di Ghidra per 8051 NON parte da sola. Serve prima **disassemblare manualmente il primo byte a CODE:0000 (tasto D)**, solo dopo l'auto-analyze trova automaticamente le altre funzioni.

## Setup GhidraMCP (se disponibile all'AI che continua)

Se hai accesso a tool MCP con prefisso `mcp__ghidra__*`, sono collegati a un'istanza Ghidra via bridge (`bridge_mcp_ghidra.py`, porta 8081). Tool utili: `list_functions`, `decompile_function_by_address`, `get_xrefs_to`, `get_xrefs_from`, `rename_function_by_address`, `set_decompiler_comment`, `list_strings`, `list_segments`. **Nota**: questi tool operano sul programma Ghidra correntemente aperto/attivo — se ci sono più programmi (uno per banco), bisogna assicurarsi che l'utente abbia il programma giusto aperto in foreground prima di interrogare.

## Convenzione di lavoro stabilita (seguila)

**Ogni volta che capisci qualcosa sul codice — anche se poi risulta sbagliato — aggiungi un commento in Ghidra** (`set_decompiler_comment`) spiegando cosa pensi che faccia la funzione E il livello di certezza (confermato vs ipotesi). Se capisci la funzione, **rinominala** (`rename_function_by_address`) con un nome sensato. Questo serve a costruire una mappa cumulativa del codice, anche i tentativi falliti sono utili da registrare.

## Fatto confermato con test hardware (2026-07-04) — LEGGI PRIMA DI CONTINUARE

Test dell'utente sul monitor fisico: con **solo cavo HDMI collegato (niente USB-C)** il monitor parte comunque a ~200% di luminosità. Senza nessun cavo va in standby dopo la scritta "D1" (normale). Questo **esclude completamente** che il bug dipenda da USB-C/DDC-CI/laptop — è un bug **puramente interno al firmware del monitor**, scatenato dal rilevamento/lock del segnale video (HSYNC/VSYNC/signal-present).

**Implicazione per la ricerca**: concentrarsi su codice legato a "segnale video rilevato" (spesso un interrupt hardware), NON sul codice del menu OSD (già ampiamente escluso, vedi sotto). Il candidato più plausibile finora è `int0_isr_dispatcher` (Banco 0, CODE:218f) — interrupt esterno 0, controlla `DAT_EXTMEM_e6f2 >> 4` per distinguere stati (3/1 vs default). Da riguardare a fondo.

## Cosa è già stato scoperto nel Banco 0

**Il codice C decompilato vero e proprio di tutte le funzioni citate sotto è salvato in `decompiled_bank0.md`** (stessa cartella) — leggilo per il dettaglio, qui sotto solo il riassunto.

- **Vettore di reset** (CODE:0000): `LJMP 0x26EB`
- **`startup_init_and_call_main`** (0x26eb): routine di avvio standard — pulisce IDATA, pulisce XDATA da 0xe000 (4096 byte), SP=0x71, decomprime tabella dati da CODE:0x1d92, chiama 0x1183
- **`main_candidate`** (0x1183): ipotesi di main(), ma è solo un wrapper sottile — la vera logica probabilmente è tutta guidata da interrupt
- **`timer0_isr_latch_fff4`** (0x0046, vettore Timer0): copia un byte da XDATA:0xfff4 a XDATA:0xe6c5 — pattern "leggi registro hardware condiviso, salva in slot per-sorgente" visto ripetuto in altri ISR
- **`int0_isr_dispatcher`** (0x218f, vettore Interrupt Esterno 0): dispatcher che controlla `DAT_EXTMEM_e6f2 >> 4` (valori 3 o 1 vs default) — entrambi i rami convergono sulla stessa funzione nel punto già analizzato, ma **è ora il candidato PRIORITARIO** come interrupt di "segnale video rilevato" (vedi fatto hardware sotto) — approfondire cosa succede PRIMA della convergenza.
- **DEAD END #1 (Banco 0)**: famiglia di 16 funzioni "trampolino" (0x03f7, 0x0409, 0x041b, ... 0x0505, chiamano 0x2200-0x22f0) che scrivono `DAT_INTMEM_59`/`uEXTMEMffff`. Chiamata da CENTINAIA di punti sparsi — troppo ubiquo per essere specifico al display, probabile infrastruttura generica (watchdog/reentrancy/debug tag). **Escluso.**
- **DEAD END #2 (Banco 1, CONFERMATO ed ESCLUSO)**: la catena e3e2/e3e3/e3e4 (Banco1 offset locale 0x3b84-0x3bdb, rinominata `videomode_e3e3_to_e3e4_lut_writer` / `videomode_e3e2_dispatch_or_bus_write`) scrive byte (0x8C,0xC4,0xC8=200,0xCC,0x84,0xD8) su XDATA:0xE3E4 in base a indice 0x08-0x0F. **Xref decisivo**: `DAT_EXTMEM_e3e2` è letto da ~80 funzioni diverse sparse in un range enorme (CODE:0x39xx-0x66xx) — pattern da "flag globale modo video/timing", non da variabile luminosità. Codice completo in `decompiled_bank1.md`. **Non seguire oltre.**
- **DEAD END #3 (Banco 1, CONFERMATO ed ESCLUSO)**: `osd_bargraph_render_percent_to_position` (0xc07a) e `osd_bargraph_setup_wrapper` (0xc7df) — renderer generico per slider OSD (luminosità/contrasto/volume/ecc), il clamp `if(100<param_3) =100` **è corretto e funziona**. Non è la fonte del bug. Codice completo in `decompiled_bank1.md`.
- **Nessuna stringa ASCII leggibile** nel firmware — il menu OSD probabilmente usa font/bitmap, non testo semplice

## CATENA BOOT RICOSTRUITA (sessione 2026-07-04)

`startup_init_and_call_main` (0x26eb) → `main_candidate` (0x1183) → **`main_superloop`** (0xf250, Banco 0):

1. Chiama **UNA SOLA VOLTA all'avvio** `signal_lock_reinit_handler` (0xf284) → chiama `boot_time_peripheral_init_ONCE` (0xda84), che scrive: `ff0f=0, ff10=0xFF, ff11=0, ff12=0xFF, ff13=0, ff14=0xFF, ff15=0, ff16=0xFF, ff17=0`.
2. Poi loop **INFINITO** (il check di uscita `FUN_CODE_feda` = solo `SETB C;RET`, sempre vero — il "break" che sembrava condizionale in realta' non scatta MAI) di ~13 chiamate alla famiglia dead-end #1 (tick generico, gia' escluso, non specifico al display).

## DEAD END #4 — CONFERMATO ED ESCLUSO CON TEST EMPIRICO SU HARDWARE (2026-07-04)

L'ipotesi "ff10/12/14/16 = PWM backlight" (candidato principale della sessione precedente) e' stata **testata fisicamente e SMENTITA**: patch di prova (4 byte da 0xFF a 0x82, vedi `patches/firmware_patch_test1_ff10_12_14_16_to_0x82.bin`) flashata sul monitor reale via CH341A. **Nessun cambiamento nella luminosita' di avvio** (rimasta identica). Xref esteso a TUTTI gli 8 banchi con codice (0,1,2,3,4,5,7,13): nessun altro scrittore/lettore di questi 4 registri da nessuna parte. **Non seguire oltre questa pista.**

Dato utile emerso dal test: la luminosita' di avvio e' **esattamente 193 (0xC1)**, non "~200" come stimato a occhio in precedenza.

## PISTA ATTUALE PIU' PROMETTENTE: DAT_EXTMEM_e65f (Banco 0)

Cercando il valore 193 (0xC1) nel firmware, trovata `DAT_EXTMEM_e65f` — quasi certamente la vera variabile di luminosita'/backlight, confermata da 3 percorsi indipendenti coerenti tra loro:

- **`FUN_CODE_e4a3`/`FUN_CODE_b5a8`** (Banco 0): funzioni "step coarse" (probabile telecomando/OSD tasto su/giu'), tabella a incrementi di 12: `0x91,0x9d,0xa9,0xb5,0xc1(=193!),0xcd,0xd9,0xe5...` con clamp superiore per livello (es. "se e65f>0xcc, e65f=0xc1" ecc — logica di step con overflow guard, sembra corretta).
- **`FUN_CODE_c626`** (Banco 0, body CODE:c626-c81e): tabella "fine" indice→valore con step ~5, stessa famiglia di valori (0,5,10,15,20,24,29,...fino a 0xa9 e oltre, funzione lunga probabilmente arriva anch'essa fino a 0xc1 e oltre — non ancora letta per intero). Chiamata da un solo punto: `FUN_CODE_b224` (CODE:b224), un decoder di protocollo seriale/IR (loop di parsing bit con timing), a sua volta chiamato da **22 punti diversi** nel codice — sembra il decoder generico di comandi da telecomando/tastiera.
- **`FUN_CODE_4536`/`4537`** (Banco 2): funzione "reset ai default" — imposta `e65d=0, e65e=0x17, e65f=0x91(145)`. Chiamata SOLO da 3 funzioni che sembrano schermate di menu OSD (lingua/reset), quindi e' un'azione utente esplicita (menu "Factory Reset"), NON qualcosa che gira automaticamente al boot. Il default di reset e' 145, non 193 — quindi NON e' la fonte diretta del valore osservato.
- **`startup_init_and_call_main`** (Banco 0, la routine di avvio hardware) pulisce XDATA da 0xe000 per 4096 byte — questo range COPRE 0xE65F (0xe000+0x65f), quindi subito dopo il reset hardware **e65f parte a 0x00**, non 193 e non 145.

**Conclusione/domanda aperta**: 193 deve essere scritto in e65f da qualche altra parte DURANTE il boot (dopo il clear XDATA, prima che l'utente veda lo schermo), non ancora trovata in nessuno degli 8 banchi scansionati. Ipotesi piu' probabile: lettura da NVRAM/EEPROM persistente (settore dedicato nella stessa flash SPI non ancora identificato con certezza — i banchi "vuoti" 6,8-12,14,15 sono stati verificati byte-per-byte come genuinamente 0xFF ovunque, quindi se c'e' un settore NVRAM vivo dovrebbe essere DENTRO uno dei banchi 0-5,7,13 gia' scansionati, non ancora individuato come tale — oppure su chip esterno separato mai confermato, es. A1315E/ET6329 visti sulla scheda ma di funzione sconosciuta).

**Non ancora trovato**: dove/se e65f viene letto per pilotare l'hardware reale (registro PWM/backlight) — nessun lettore trovato oltre alle funzioni di modifica stesse in Banco 0/2. Il "consumo finale" verso l'hardware deve essere altrove.

## Perche' potrebbe NON essere un bug di clamp mancante

Se e65f e' davvero un valore "impostazione utente" persistito ma che si RESETTA ad ogni accensione invece di essere ricaricato dal salvataggio precedente, il sintomo "parte sbagliato ogni volta, l'utente lo abbassa, ma al riavvio torna alto" e' perfettamente coerente con: **la routine di ripristino impostazioni da NVRAM manca, e' rotta, o non viene mai chiamata** — non necessariamente un clamp percentuale mancante. Cercare quindi una funzione "load settings"/"restore from NVRAM" chiamata durante il boot (prima o dopo signal_lock_reinit_handler), non ancora identificata.

## Prossimi passi concreti (aggiornati dopo test hardware fallito su ff10/12/14/16)

1. **PRIORITARIO**: trovare dove/se esiste una routine di lettura NVRAM/EEPROM chiamata durante il boot (prima che main_superloop entri nel loop infinito) — cercare pattern di accesso seriale/I2C bit-bang (toggle ripetuto di 2 pin SFR in sequenza), o xref a un eventuale registro "EEPROM data/ready" non ancora identificato.
2. Leggere per intero `FUN_CODE_c626` (troncata nella lettura precedente a causa della lunghezza, CODE:c626-c81e in Banco0) per vedere se la tabella indice→valore arriva fino a un indice che produce ESATTAMENTE 193 — confermerebbe che l'indice/tasto usato per "aumenta luminosita'" numerato produce quel valore specifico, utile per capire se 193 e' un valore "raggiungibile normalmente" o anomalo.
3. Investigare `FUN_CODE_b224` (il decoder IR/seriale) piu' a fondo — capire quale protocollo decodifica, e se tra i 22 chiamanti ce n'e' uno legato al boot/power-on piuttosto che a input utente runtime.
4. Cercare in Banco 0 xref a `DAT_EXTMEM_e65d`/`DAT_EXTMEM_e65e` (le due variabili "sorelle" di e65f, stessa famiglia, probabilmente contrasto/altro parametro) per contesto aggiuntivo su come viene gestito l'intero gruppo di impostazioni.
5. Una volta identificato il bug con certezza: proporre la patch binaria minima, NON scrivere subito sulla flash reale — prima discuti la patch con l'utente. Setup di test collaudato: patch su copia in `patches/`, flash con `flashrom -p ch341a_spi -w`, test su hardware reale, rollback pronto con `dumps/firmware_dump_1.bin` (hash verificato).

## AGGIORNAMENTO FINALE SESSIONE (2026-07-04) — entrambe le patch fisiche testate hanno fallito

- Patch #1 (ff10/12/14/16 → 0x82): nessun effetto su hardware reale
- Patch #2 (forza e65f=145 in signal_lock_reinit_handler): nessun effetto, luminosità rimasta 193 esatto
- Confermato: main_superloop esegue davvero (verificato byte-level il trucco PUSH+RET usato per raggiungerlo)
- Tutti i 6 vettori di interrupt tracciati, nessuno tocca e65f
- Trovato protocollo servizio/calibrazione seriale (FUN_CODE_22fb/23f1, dispatcher 0x209b-0x20b0) — probabile UART factory-test, non legato al boot
- bus_write (ff60/61/64/65) confermato dead-end definitivo per luminosità (centinaia di occorrenze, sempre legate a e3e2/e3e3/e3e4 video-mode)

**Conclusione onesta**: ricerca statica pura ha raggiunto un limite pratico senza debugger hardware. Ipotesi rimaste: (1) l'hardware del RTD2525BE carica la luminosità da un meccanismo interno indipendente da e65f/RAM firmware, (2) esiste un percorso non ancora scoperto raggiungibile solo tramite bank-switching non decodificato. Vedi `notes.md` sezione "Stato al termine sessione" per dettagli completi e prossimi passi consigliati (oscilloscopio/logic analyzer consigliato come prossimo passo più concreto).

## Fatto hardware confermato che ha riorientato la ricerca (2026-07-04)

Bug si verifica anche con SOLO HDMI (no USB-C) → puramente firmware-interno, scatenato dal boot/lock segnale video, non da DDC-CI/host.

## Sicurezza

- Il backup del firmware originale esiste ed è verificato (`firmware_dump_1.bin` / `firmware_dump_2.bin`, hash identici)
- Qualsiasi patch va testata partendo da una COPIA del dump, mai sovrascrivere i file originali in questa cartella
- Prima di riscrivere la flash fisica (via `flashrom -p ch341a_spi -w`), l'utente deve confermare esplicitamente — non farlo in autonomia
