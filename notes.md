# Monitor D1 — Reverse Engineering firmware

## Problema originale
Due bug osservati sul monitor "D1" (no-name/OEM, collegato via USB-C al laptop TUXEDO InfinityBook S Gen8):
1. Il PC non si carica dall'USB-C del monitor (nonostante il monitor dovrebbe poterlo fare)
2. Il monitor parte a ~200% di luminosità all'accensione, poi va abbassata manualmente a 100%

## Hardware identificato sulla scheda del monitor

| Chip | Ruolo | Note |
|---|---|---|
| **RTD2525BE** (Realtek, serie LRH) | Scaler video, pilota pannello LVDS | QFN, "no external memory required" per il video, ma probabile core 8051 interno per OSD/gestione sistema |
| **CPW3220** (Chipsea) | Controller USB-C PD | **ARM Cortex-M0**, stessa famiglia di CS32G020/1 (datasheet ufficiale letto). Gestisce probabilmente il bug di carica USB-C |
| **GL850G** (Genesys Logic) | Hub USB 2.0 | Non rilevante per i bug, gestisce solo le porte USB-A passthrough |
| **ZB25VQ80** (Zbit Semiconductor) | Flash SPI esterna, 1MB, chip 3V | **Dumpata con successo**, contiene firmware 8051 genuino |
| A1315E | Non identificato | Probabile componente di supporto minore |
| ET6329 | Non identificato | Probabile componente di supporto minore |

## Bug di carica USB-C (CPW3220) — diagnosticato, non ancora risolto

- Trovato `power_role` bloccato su `source` invece di `sink` — risolto con stacca/riattacca cavo (bug di negoziazione UCSI/kernel)
- Dopo il fix: negoziazione PD resta fissa a 250mA (~1-2W), mai completa il contratto di potenza vero
- Causa probabile: il CPW3220 non comunica correttamente i PDO (Power Data Objects) — coerente con errore kernel visto nei log: `ucsi_acpi: GET_CONNECTOR_CAPABILITY failed (-110)`
- Il CPW3220 supporta aggiornamento firmware via: (1) USB-C diretto con tool Chipsea proprietario, (2) SWD (ARM standard, serve ST-Link economico), (3) interfaccia seriale proprietaria (PCL/PDA)
- **Non ancora tentato un dump/analisi del firmware CPW3220** — resta da fare se si vuole risolvere questo bug specifico

## Dump firmware flash ZB25VQ80 — completato

- Hardware: CH341A (variante ZIF nera) + clip SOP8 + adattatore SOIC8→DIP8
- Il CH341A di Carlo era stato modificato in passato per output 3.3V fisso (seguendo una guida di modding) — poi **ripristinato a 5V** per lavoro precedente sull'header I2C del monitor (non serviva per questo dump, che è correttamente a 3.3V per lo ZB25VQ80)
- Jumper CH341A: quello a 3 pin (posizioni 1-2/2-3) seleziona la **modalità** (UART vs SPI/I2C via USB PID 5523 vs 5512), non il voltaggio come su altre varianti — occhio a non confondersi
- Procedura di lettura: `flashrom -p ch341a_spi -r dump.bin` (serve scaricare eventuali moduli kernel `ch341*` prima, altrimenti flashrom non trova il chip)
- **Due dump letti, hash SHA256 identici** → lettura stabile e affidabile
- File: `dumps/firmware_dump_1.bin`, `dumps/firmware_dump_2.bin` (1.048.576 byte, 66.5% vuoto/0xFF)

## Analisi firmware in Ghidra

- Linguaggio: **8051:BE:16:default** (big-endian, confermato dai byte del vettore di reset)
- **GhidraMCP** configurato e funzionante: bridge Python (`bridge_mcp_ghidra.py`) in ascolto su `127.0.0.1:8081` (SSE), registrato in Claude Code come server MCP scope "user" col nome `ghidra`. Il plugin lato Ghidra ascolta su porta 8080.
- L'auto-analisi di Ghidra per 8051 non parte da sola sul binario raw — serve **disassemblare manualmente il primo byte a CODE:0000 (tasto D)** prima che l'auto-analyze trovi le altre ~600 funzioni
- Vettore di reset (0x0000): `LJMP 0x26EB`
- **FUN_CODE_26eb**: routine di avvio standard — pulisce IDATA, pulisce XDATA a partire da 0xe000 (4096 byte), imposta SP=0x71, decomprime una tabella di inizializzazione dati da CODE:0x1d92, poi chiama FUN_CODE_1183
- **FUN_CODE_1183 → FUN_CODE_03f7 → FUN_CODE_2200(0x22)**: catena di init minimale (resetta due variabili, es. `DAT_INTMEM_59` e `uEXTMEMffff`), sembra un modulo/subsystem specifico, non necessariamente il main loop vero
- **Interrupt Timer0 (0x0046)**: copia un byte da `XDATA:0xfff4` a `XDATA:0xe6c5` — pattern "leggi registro hardware vicino al top dello spazio XDATA, salva in variabile" visto ripetuto in molti punti del codice (probabile lettura di stato/registri periferici)
- **Nessuna stringa ASCII leggibile** nel firmware (né con `strings` né con l'analisi di Ghidra) — il menu OSD probabilmente usa font/bitmap, non testo semplice

## Fatto confermato importante (test utente 2026-07-04)

- Monitor senza NESSUN cavo: mostra solo scritta "D1" poi va in standby (normale, nessun segnale)
- Monitor con SOLO cavo HDMI (niente USB-C): **parte comunque a ~200% di luminosità**
- Conferma: il bug è interamente interno al firmware del monitor (RTD2525BE), si attiva al momento del lock/rilevamento segnale video. NON dipende da USB-C, DDC-CI, o dal laptop/driver Linux — possiamo escludere completamente il lato host.
- Implicazione per la ricerca: il bug è probabilmente nella routine "on signal detected/lock" (spesso guidata da interrupt hardware tipo INT0 per HSYNC/VSYNC/signal-present), NON nel codice del menu OSD (che finora abbiamo escluso ampiamente). Rivalutare int0_isr_dispatcher (CODE:218f) e la sua catena come possibile trigger.

## Catena boot ricostruita (sessione 2026-07-04) — CANDIDATO PRINCIPALE per il bug

`startup_init_and_call_main` (0x26eb) → `main_candidate` (0x1183, carica DPTR=0xf250 poi salta nel dispatcher generico) → **`main_superloop`** (0xf250, ex "signal_acquire_retry_loop", rinominato dopo scoperta che il "retry" non scatta mai — vedi sotto):

1. Chiama **UNA SOLA VOLTA all'avvio** `signal_lock_reinit_handler` (0xf284), che a sua volta chiama:
   - **`boot_time_peripheral_init_ONCE`** (0xda84) — reset hardware massiccio (timer, interrupt) e scrive: `ff0f=0, ff10=0xFF, ff11=0, ff12=0xFF, ff13=0, ff14=0xFF, ff15=0, ff16=0xFF, ff17=0`. **CANDIDATO PRINCIPALE PER IL BUG 200%**: pattern a 4 coppie di registri consecutivi, alternato 0/0xFF(max). Xref conferma: nessun'altra funzione in Banco0 scrive MAI questi 4 registri (ff10/12/14/16) dopo questo punto. Se anche uno solo è il duty/PWM del backlight, resta bloccato al valore "provvisorio" 0xFF finché l'utente non lo cambia a mano da OSD.
   - `FUN_CODE_e9d7` — costruisce un record leggendo parametri hardware (ipotesi: misura caratteristiche segnale appena rilevato)
2. Poi entra in un **loop infinito** (il controllo di uscita `FUN_CODE_feda` = solo `SETB C; RET`, sempre vero, quindi il "break" non scatta mai) che esegue ~13 chiamate alla famiglia dead-end #1 (scheduler/tick generico, non specifico al display)

**Non ancora trovato**: una funzione che legga impostazioni salvate (EEPROM/NVRAM/default) e sovrascriva ff10/12/14/16 con il valore corretto DOPO l'init. Se non esiste, o non viene mai chiamata, potrebbe essere direttamente la causa del bug (manca "applica luminosità salvata").

**Prossimo passo consigliato**: o (a) continuare la ricerca statica per trovare/escludere una routine "restore settings" mancante, oppure (b) test empirico su hardware — patchare uno dei 4 byte (es. ff10 o ff12, i più probabili target "duty cycle") a un valore più basso (es. 0x64 o 0x32) e riflashare per vedere se cambia la luminosità di avvio. **Da NON fare senza discuterne prima con l'utente e senza backup verificato** (già disponibile).

## Cosa NON abbiamo ancora trovato

- L'esatta routine che imposta la luminosità di avvio (il bug dei "200%")
- Se il bug è nel firmware RTD2525BE (questo dump) o in un percorso diverso
- Il registro/indirizzo esatto che controlla il PWM della retroilluminazione

## Strategia di modifica pianificata (quando troveremo il bug)

**Non si ricompila da zero** — non abbiamo compilatore/SDK/header originali, sarebbe come riscrivere tutto il firmware. Si farà invece una **patch binaria mirata**: individuare i pochi byte responsabili (es. una costante sbagliata o un salto), modificarli direttamente nel `.bin`, riflashare con lo stesso setup CH341A+clip già validato, testare, con backup del dump originale sempre disponibile come rete di sicurezza.

## Struttura cartelle progetto

```
~/Code/Realtek/MonitorD1/
├── dumps/              — backup firmware letti (2 dump identici, verificati)
├── ghidra_project/      — dove salvare il progetto .gpr di Ghidra (rename/commenti persistenti)
├── patches/             — versioni patchate del firmware (quando pronte)
├── notes.md             — questo file
├── RTDMultiProg/         — tool Python per chip RTD2660/2662 (NON usato alla fine, RTD2525BE non ha ISP flash)
└── ch341-i2c-spi-gpio/   — driver kernel I2C per CH341A (usato per il lavoro sull'header ISP del monitor, non per il dump SPI)
```

## Handoff per altro modello AI

Creato `ghidra_code/AI_HANDOFF_PROMPT.md` — prompt completo e autosufficiente per un altro modello AI che continui il reverse engineering (contesto hardware, struttura banchi, tabella di import, cosa già scoperto/escluso, prossimi passi). Copie del dump firmware anche lì dentro (`ghidra_code/firmware_dump_1.bin`, `firmware_dump_2.bin`).

## Patch di test #1 (2026-07-04) — verifica ipotesi ff10/12/14/16

File: `patches/firmware_patch_test1_ff10_12_14_16_to_0x82.bin` (copia di `firmware_dump_1.bin`, originale intatto).

4 byte modificati (tutti da 0xFF a 0x82, ~51% se scala 0-255):
| Registro | Offset file | Vecchio | Nuovo |
|---|---|---|---|
| ff10 | 0xdb1b | 0xFF | 0x82 |
| ff12 | 0xdb26 | 0xFF | 0x82 |
| ff14 | 0xdb31 | 0xFF | 0x82 |
| ff16 | 0xdb3c | 0xFF | 0x82 |

Contesto: questi byte sono l'operando immediato di "MOV A,#0xFF" dentro `boot_time_peripheral_init_ONCE` (CODE:0xda84, Banco 0), scritti una sola volta all'avvio su 4 registri XDATA consecutivi (vedi `decompiled_bank0.md`). Nessun'altra funzione in tutti gli 8 banchi con codice (0,1,2,3,4,5,7,13) scrive mai questi 4 registri altrove — ipotesi: se uno è il duty/PWM del backlight, resta fisso a 0xFF (max) per tutta l'esecuzione, causando il bug "200%".

**Risultato atteso se ipotesi corretta**: il monitor parte visibilmente meno luminoso (circa metà) invece che troppo luminoso.
**Risultato atteso se ipotesi sbagliata**: nessun cambiamento visibile — si scarta la pista, si torna alla ricerca statica (es. dubbio ff0a come registro di soglia, non output).

**Procedura di flash**: stesso setup CH341A + clip SOP8 già validato. `flashrom -p ch341a_spi -w patches/firmware_patch_test1_ff10_12_14_16_to_0x82.bin`. Backup verificato disponibile per rollback immediato (`dumps/firmware_dump_1.bin`, hash `43299e11...`).

## Variabile luminosità identificata: DAT_EXTMEM_e65f (sessione 2026-07-04, dopo test patch fallito)

Il test patch #1 (ff10/12/14/16) NON ha funzionato — luminosità di avvio rimasta a **193 esatto** (non ~200 stimato). Cercando 193 (0xC1) nel firmware:

- **`DAT_EXTMEM_e65f`** (Banco 0) è quasi certamente la vera variabile di luminosità/backlight. Confermato da 3 percorsi indipendenti che la modificano con logica coerente da "impostazione utente":
  - `FUN_CODE_e4a3`/`FUN_CODE_b5a8` (Banco 0): step "coarse" da telecomando/OSD, incrementi di 12 (0x91,0x9d,0xa9,0xb5,**0xc1**,0xcd,0xd9,0xe5...) con clamp superiore per livello
  - `FUN_CODE_c626` (Banco 0): tabella "fine" indice→valore, step ~5, stessa famiglia di valori
  - Chiamata da `FUN_CODE_b224` (Banco 0): decoder di un protocollo seriale/IR (loop di parsing bit con timing), chiamato da 22 punti diversi nel codice — sembra il decoder generico di comandi remoti
- **`FUN_CODE_4536`/`4537`** (Banco 2): funzione "reset ai default" — imposta `e65d=0, e65e=0x17, e65f=0x91(145)`. Chiamata SOLO da 3 funzioni che sembrano schermate menu OSD (lingua/reset), NON automaticamente al boot. Default reset = 145, non 193 — quindi non è la fonte diretta del bug.
- **`startup_init_and_call_main`** (Banco 0, boot) pulisce XDATA da 0xe000 per 4096 byte — questo range COPRE 0xe65f, quindi a freddo (subito dopo il reset hardware) e65f parte a **0x00**, non 193.
- **Conclusione**: 193 deve essere caricato in e65f da qualche altra parte durante il boot, non ancora trovata — probabile lettura da NVRAM/EEPROM persistente (interna alla stessa flash SPI in un settore non ancora identificato, o su chip esterno separato, es. A1315E/ET6329 mai confermati). Nessuna funzione "restore/load settings" trovata finora in nessuno degli 8 banchi scansionati (0,1,2,3,4,5,7,13).
- **Non ancora trovato**: dove/se e65f viene letto per pilotare l'hardware reale (PWM/backlight) — nessun lettore trovato oltre alle funzioni di modifica stesse in Banco 0. Il "consumo" finale deve essere in un banco non ancora scansionato a fondo per questo specifico xref, o raggiunto via indirect/bank-switch non tracciato.

## Patch di test #2 (2026-07-04) — code cave, forza e65f=145 al boot

Dopo aver escluso ff10/12/14/16 (test #1 fallito) e tracciato TUTTI i 6 vettori di interrupt + l'intera catena di boot senza trovare chi scrive 193 in e65f (probabile auto-load hardware, ipotesi utente), approccio diverso: invece di cercare il "pezzo mancante", **iniettiamo noi un'istruzione** che forza il valore corretto subito prima che il loop principale parta, sovrascrivendo qualsiasi default hardware.

File: `patches/firmware_patch_test2_force_e65f_145.bin` (copia di `firmware_dump_1.bin`, originale intatto).

**Meccanismo (nessun byte spostato, tutti gli indirizzi/salti del resto del firmware restano validi)**:
- Offset `0xf2ae` (dentro `signal_lock_reinit_handler`, Banco 0, chiamata una sola volta all'avvio): 3 byte `90 e6 e0` (MOV DPTR,#0xe6e0) sostituiti con `12 29 2b` (LCALL 0x292b)
- Code cave a offset `0x292b` (328 byte liberi 0xFF trovati in Banco0, area completamente inutilizzata): scritti 10 byte
  ```
  90 e6 5f    MOV DPTR,#0xe65f
  74 91       MOV A,#0x91 (145 decimale)
  f0          MOVX @DPTR,A        -> e65f = 145
  90 e6 e0    MOV DPTR,#0xe6e0    (ripristina l'istruzione tolta dall'hook, il codice originale prosegue identico)
  22          RET
  ```

**Perché 145 (0x91)**: è il valore usato dalla funzione di "reset impostazioni a default di fabbrica" del firmware stesso (`FUN_CODE_4536`/`4537`, Banco 2) — non un numero scelto a caso, ma il default che il firmware considera corretto.

**Perché questo punto di aggancio**: `signal_lock_reinit_handler` gira una sola volta, il più tardi possibile prima che main_superloop entri nel loop infinito — se c'è un auto-load hardware che scrive 193 PRIMA di questo punto, lo sovrascriviamo qui; se l'auto-load avviene DOPO, servirà un punto di aggancio più tardivo (da valutare se questo primo tentativo non funziona).

**Risultato atteso se funziona**: il monitor parte a luminosità ~145 (livello "factory default"), non più a 193.
**Se non cambia nulla**: l'auto-load hardware (se esiste) avviene dopo questo punto nel tempo — serve agganciare più avanti nell'esecuzione (es. dentro il loop infinito di main_superloop, alla prima iterazione).

## Stato al termine sessione 2026-07-04 — bilancio onesto

**Testato fisicamente su hardware reale, ENTRAMBI falliti (nessun cambiamento visibile)**:
- Patch #1: forzare ff10/12/14/16 (0xFF→0x82) — nessun effetto
- Patch #2: forzare e65f=145 al boot (code cave in signal_lock_reinit_handler) — nessun effetto, luminosità rimasta 193 esatto

**Confermato con certezza (analisi statica + verifica byte-level)**:
- `main_superloop` (0xf250) esegue davvero al boot (confermato il trucco PUSH+RET che lo raggiunge da state_setter_state00)
- Tutti i 6 vettori di interrupt tracciati (INT0, INT1, Timer0/1/2, Seriale) — nessuno scrive e65f
- Trovato un vero protocollo seriale a pacchetti (checksum, lunghezza, lettura/scrittura memoria arbitraria via puntatore) in `FUN_CODE_22fb`/`FUN_CODE_23f1` (Banco 0, dispatcher a 0x209b-0x20b0) — sembra un servizio di fabbrica/calibrazione via UART, non legato al boot automatico
- Pista "bus_write" (ff60/61/64/65) confermata legata solo a parametri video (e3e2/e3e3/e3e4), centinaia di occorrenze, MAI e65f — dead end confermato definitivamente
- Nessuna assegnazione hardcoded dedicata "= 0xC1(193)" trovata in nessun banco al di fuori della tabella e65f già nota

**Conclusione**: la ricerca statica pura (senza debugger hardware/logic analyzer) ha raggiunto un limite pratico. Le spiegazioni rimaste plausibili:
1. L'hardware del chip video (RTD2525BE) carica la luminosità da un meccanismo interno (registro/EEPROM proprietario) completamente indipendente dalla RAM del firmware che abbiamo analizzato — e65f potrebbe essere una variabile "shadow" per la sola visualizzazione OSD, scollegata dal vero registro di comando
2. Esiste un percorso di codice non ancora scoperto, forse raggiunto tramite un meccanismo di cambio-banco (bank-switching) che non abbiamo ancora decodificato, o in una delle zone "orfane" non analizzate rimaste (es. jump table di FUN_CODE_a1f7, mai risolta)

**Prossimi passi consigliati** (in ordine di costo crescente):
1. Se disponibile, un oscilloscopio/logic analyzer sul pin PWM del backlight per correlare in tempo reale i cambi di stato con l'accesso a registri — l'unico modo per confermare/escludere l'ipotesi hardware-autonomo con certezza
2. Decodificare il meccanismo di bank-switching dell'8051 (come il firmware passa da un banco all'altro) — permetterebbe di verificare xref cross-bank che finora non abbiamo potuto tracciare
3. Terzo tentativo di patch (se si vuole rischiare ancora hardware): agganciare il force-write DENTRO il loop infinito di tick (non solo una volta all'ingresso) per escludere che qualcosa lo sovrascriva continuamente — ma nessuna prova concreta supporta questa ipotesi specifica al momento

## Sessione Fable 5 (2026-07-04) — bank-switch DECODIFICATO + correzioni importanti

### 1. Meccanismo di bank-switching — RISOLTO (era il lead #1 del handoff)
- **XDATA `0xFFFF` = registro hardware di bank-select.** Scrivendo il numero di banco (0-15) lì, il chip mappa la pagina flash da 64KB corrispondente nello spazio codice.
- `DAT_INTMEM_59` (IDATA 0x59) = shadow del banco corrente (nibble basso).
- Famiglia di **gate** a 0x2200/0x2210/0x2220/0x2230/0x2240/0x2250 (una per banco 0..5): fanno `MOV DPTR,#0xffff; MOV 0x59,#N; MOVX @DPTR,A` → selezionano il banco N.
- Famiglia di **trampolini** 0x03f7/0x0409/.../0x0451: leggono 0x59 (nibble basso = banco corrente), fanno PUSH del banco + PUSH DPTR(target), poi `LJMP` a un gate → **chiamata cross-bank via RET-to-DPTR**. Salvano il banco chiamante sullo stack per ripristinarlo al ritorno.
- Rinominati/commentati in Ghidra (Banco0): `bankgate_select_bank0_HWREG_ffff` (0x2200), `bankgate_select_bank5_HWREG_ffff` (0x2250), commenti su 0x1087/0xb773/0x1d92.
- **Nota**: la famiglia 0x2200 NON era un "watchdog/state tag generico" come ipotizzato prima — è il call-gate cross-bank. Rivalutare quella conclusione precedente.

### 2. CORREZIONE: e65f al boot vale 145, NON 0
- Il decompressore XINIT a `CODE:0x1d92` (in `startup_init_and_call_main`) gira DOPO il clear di e000-efff e scrive i globali inizializzati. Decodificata la tabella: imposta **e65d=0x00, e65e=0x17, e65f=0x91 (145)**.
- Quindi e65f parte a **145** (= factory default), non a 0 come diceva una nota precedente.
- **Conseguenza pesante**: la patch #2 (forzare e65f=145 al boot) era un **NO-OP** — scriveva 145 su un valore già 145. Quel test "fallito" **non dimostra niente** sul fatto che e65f piloti o meno la retroilluminazione. La conclusione "e65f scollegato dall'hardware" è NON supportata da quel test.
- Nessuna voce della tabella XINIT, e nessun assegnamento hardcoded sul percorso di boot, vale **0xC1 (193)**. 193 compare SOLO come "livello 7" nelle tabelle a step OSD di e65f (0x91,0x9d,0xa9,0xb5,**0xc1**,0xcd,...). Quindi 193 non è un default statico: o è scritto a runtime (chiamando lo step-setter con livello 7), o è il default di power-on dell'hardware, o è mostrato da un registro hardware letto dall'OSD.

### 3. Consumer di e65f tracciato (ma probabilmente è RENDERING OSD, non backlight)
- Catena: `e65f → e3d0` (buffer apply, via helper b773/b7b6/b7f9/b83c/b8c2) `→ FUN_CODE_1087 → gate 0451 → BANCO 5 → FUN_CODE_38e3 → FUN_CODE_130a`.
- MA `FUN_CODE_130a` (Banco5) è un **writer di memoria indicizzato generico**: nel loop di 38e3 scrive il byte del registro `ff7b` in un buffer all'indirizzo `[e3cf:e3d0]`, usando il valore come **indice di indirizzo**. Pattern tipico di **rendering glifo/framebuffer OSD** (disegna il numero sullo schermo), NON scrittura di un duty PWM. Registri coinvolti: `ff78-ff7c` = engine di trasferimento (probabile I2C master o blitter OSD), non PWM backlight.
- Quindi: il consumer di e65f che ho trovato è quasi certamente il percorso di **visualizzazione** del valore. e65f resta plausibilmente una variabile shadow OSD. Il vero registro PWM backlight NON è ancora identificato — ma ora ho la chiave (bank-switch) per tracciare xref cross-banco verso di esso.

### 4. Ipotesi bug aggiornata (la più coerente con TUTTI i sintomi)
Il valore di backlight non viene MAI applicato all'hardware al boot: la catena di apply gira solo quando l'utente cambia il valore (tasto Minus/OSD). Al power-on il driver LED sta al suo default hardware (luminoso, letto ~193). e65f viene re-inizializzato a 145 ad ogni boot (mai salvato in NVRAM) e mai auto-applicato. → ogni accensione = luminoso finché non si tocca a mano. **Corollario nuovo e importante**: una patch efficace deve **triggerare la catena di apply/dispatch**, non solo settare la variabile (motivo aggiuntivo per cui patch #2 non poteva funzionare).

### 5. Prossimi passi raccomandati (Fable 5)
1. **Test decisivo, 1 byte**: cambiare il byte XINIT di e65f a **offset file `0x1f85`** (attuale `0x91`) in un valore molto diverso, es. `0x30`, riflashare, osservare: (a) se lo slider OSD backlight legge ~0x30 → e65f È il valore OSD; (b) se la luminosità fisica cambia → e65f pilota l'hardware; (c) se niente cambia → e65f è puro shadow e 193 è default hardware. Distingue le 3 ipotesi che i due test precedenti (entrambi viziati) non hanno potuto separare.
2. Ora che il bank-switch è decodificato: cercare cross-banco (specialmente Banco1/5) un vero registro PWM (coppia periodo/duty) scritto con un valore scalato, distinto dal percorso di rendering OSD ff78-7c.
3. Riconsiderare se lo slider "Backlight" del menu OSD legge davvero e65f o un registro hardware (spiegherebbe perché mostra 193 mentre e65f=145 al boot).

## TODO

- [ ] Salvare il progetto Ghidra corrente dentro `ghidra_project/` (invece che nella posizione di default) così rename/commenti non si perdono
- [ ] Tracciare gli altri vettori di interrupt (INT1 @0x0013, Timer1 @0x001B, Serial @0x0023, Timer2 @0x002B) cercando pattern PWM/duty-cycle
- [ ] Cercare xref verso indirizzi XDATA sospetti (vicino a 0xfff4, 0xe6xx) per capire la mappa registri hardware
- [ ] Rinominare progressivamente solo le funzioni/variabili nel percorso che porta alla luminosità (non tutte le 600)
- [ ] Una volta trovato il bug: preparare patch minimale, documentarla, testare su hardware con backup pronto
- [ ] (Secondario, non urgente) Eventualmente investigare firmware CPW3220 per il bug di carica USB-C — richiede SWD/ST-Link, non ancora iniziato
