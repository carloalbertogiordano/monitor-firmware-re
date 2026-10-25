# Prompt per Fable 5 — trova il bug di luminosità nel firmware del monitor "D1"

Incolla questo prompt per intero. Il file firmware completo (1MB) è a: `~/Code/Realtek/MonitorD1/dumps/firmware_dump_1.bin` (hash SHA256: `43299e119999e39c1f26185674ad71ff854ccc98e4eceae45ddbf8c3a0976d8b`, verificato con doppia lettura identica). Trovi anche tutto il lavoro di analisi già fatto in `~/Code/Realtek/MonitorD1/` (notes.md, ghidra_code/).

---

## Obiettivo

Monitor portatile USB-C/HDMI (marchio OEM cinese, venduto come BJYP D1 / Kwumsy D1 / Hyangin D1 — stessa scheda, brand diversi). Bug: **si accende sempre a luminosità troppo alta** (letta esattamente 193 su una scala che sembra 0-255), e va abbassata manualmente ogni volta con il tasto fisico "Minus" (dedicato alla luminosità) o dal menu OSD — non resta salvata, torna a 193 a ogni riaccensione.

**Confermato con test fisico**: il bug è puramente interno al firmware del monitor. Succede anche con solo cavo HDMI collegato (niente USB-C/laptop), quindi non ha niente a che fare con DDC-CI/USB-C/host.

Non stiamo cercando di ricompilare il firmware (nessun SDK/compilatore originale disponibile) — l'obiettivo è una **patch binaria minima**: capire quali byte cambiare.

## Hardware

- Chip video/scaler: **RTD2525BE** (Realtek, core 8051 esteso, esegue questo firmware)
- Chip backlight: quasi certamente **LA1315E** (Lattice Art, regolatore step-down DC-DC 3.6-30V 5A, ESOP-8) — probabile driver di potenza del backlight LED, controllato via segnale PWM o analogico dal chip video
- Flash SPI dumpata: ZB25VQ80 (1MB, Zbit)
- UI fisica: **NON un semplice tasto potenziometro** — tasto "Minus" dedicato a "Backlight Adjustment", tasto "Plus" a "Volume", tasto "Multi-Function" per menu/conferma/ritorno OSD completo (quindi ESISTE anche un menu OSD navigabile con slider luminosità, oltre al tasto diretto)

## Struttura firmware — 8051 con bank-switching

Processore 8051 big-endian, 64KB indirizzabili per volta ma il file è 1MB → 16 banchi da 64KB, di cui hanno contenuto reale (non 0xFF vuoto): banco 0 (offset 0x000000), 1 (0x010000), 2 (0x020000), 3 (0x030000), 4 (0x040000), 5 (0x050000, parziale ~26%), 7 (0x070000, parziale ~17%, contiene molto rumore/dati misinterpretati come codice), 13 (0x0d0000, parziale ~9%, quasi vuoto). Banchi 6,8-12,14,15 sono COMPLETAMENTE vuoti (verificato byte-per-byte).

**IMPORTANTE, non ancora decodificato**: il meccanismo di bank-switching stesso (come il firmware passa da un banco all'altro durante l'esecuzione) non è mai stato reverse-engineered in questa indagine. Tutta l'analisi finora ha trattato ogni banco come Ghidra-program separato e ha fatto ricerche SOLO all'interno di ogni singolo banco — è possibile che il vero codice che risolve il bug sia raggiungibile SOLO tramite una chiamata cross-bank che non abbiamo mai tracciato. **Questa è probabilmente la pista più promettente non ancora esplorata.**

## Cosa è CONFERMATO (con verifica byte-level, fidati di questo)

- Vettore di reset (CODE:0000, Banco 0) → `LJMP 0x26EB` → `startup_init_and_call_main` (pulisce IDATA e XDATA 0xe000-0xefff, poi chiama 0x1183)
- 0x1183 (`main_candidate`) fa un trucco: `MOV DPTR,#0xf250; LJMP 0x03f7` — NON chiama direttamente 0xf250, ma tramite un meccanismo PUSH+RET (dentro 0x2200, che fa PUSH di DPTR poi RET, usando il RET come "salto calcolato") arriva davvero a eseguire 0xf250. Verificato byte per byte, CONFERMATO che esegue.
- 0xf250 (`main_superloop`, Banco 0) è la vera main loop: chiama `signal_lock_reinit_handler` (0xf284) **UNA SOLA VOLTA**, poi entra in un loop infinito che esegue ~13 chiamate a una famiglia di funzioni "dead-end" (0x2200-0x22f0, sembra un tag di stato generico/watchdog, chiamata da centinaia di punti nel codice, NON specifica al display — verificato e scartato con certezza).
- `signal_lock_reinit_handler` chiama `boot_time_peripheral_init_ONCE` (0xda84), che fa un reset hardware massiccio (timer, interrupt) e scrive 4 registri XDATA consecutivi (ff10, ff12, ff14, ff16) da 0x00/0xFF alternati — **TESTATO SU HARDWARE REALE, patchati questi 4 byte da 0xFF a 0x82, NESSUN EFFETTO sulla luminosità. Escludere questa pista.**
- Tutti i 6 vettori di interrupt (INT0 @0x0003, INT1 @0x0013, Timer0 @0x000B, Timer1 @0x001B, Timer2 @0x002B, Seriale @0x0023) sono stati tracciati — nessuno tocca la variabile candidata e65f (vedi sotto) né altri registri PWM ovvi.
- Trovata variabile `DAT_EXTMEM_e65f` (Banco 0) con forte evidenza circostanziale di essere la "luminosità": letta/scritta da 3 funzioni indipendenti coerenti (step coarse a incrementi di 12: 0x91,0x9d,0xa9,0xb5,**0xc1=193**,0xcd,0xd9,0xe5; step fine con tabella a incrementi di 5; reset-a-default che imposta 0x91=145). **TESTATO SU HARDWARE REALE**: patchato un code-cave che forza e65f=145 subito prima che main_superloop entri nel loop infinito — **NESSUN EFFETTO**, luminosità rimasta 193 esatto. La ricerca esaustiva di xref a e65f in TUTTI gli 8 banchi con codice non ha mai trovato un "consumer" che applichi questo valore a un registro hardware — quindi o (a) e65f è scollegata dal vero output hardware (variabile solo per display OSD), o (b) il vero collegamento passa per bank-switching non tracciato.
- Il meccanismo "bus_write" (registri ff60/61/64/65, funzioni `bus_write_variant_a_ff64_65`/`bus_write_variant_b_ff60_61` in Banco 1) è usato CENTINAIA di volte ma sempre legato a `DAT_EXTMEM_e3e2/e3e3/e3e4` (un flag "modo video/timing", confermato da xref a ~80 funzioni diverse) — **dead end confermato definitivamente per la luminosità**, non seguire.
- Trovato un vero protocollo di servizio seriale (probabile UART factory-test/calibrazione) in Banco 0: `FUN_CODE_22fb` (legge memoria arbitraria, con checksum) e `FUN_CODE_23f1` (scrive memoria arbitraria), dispatcher a 0x209b-0x20b0. Non ancora collegato al boot automatico — sembra richiedere trigger esterno via UART.
- Il tasto fisico "Minus" (backlight) probabilmente non passa per un decoder IR/telecomando come inizialmente ipotizzato (`FUN_CODE_b224`, un parser di protocollo seriale con timing bit-a-bit) — riconsiderare se invece è un semplice polling GPIO su un pin pulsante, dato che il manuale conferma tasti fisici dedicati, non un telecomando IR.

## Convenzione di lavoro (segui questa)

**Ogni volta che scopri qualcosa sul codice — anche se poi risulta sbagliato — aggiungi un commento in Ghidra** (`set_decompiler_comment`) spiegando l'ipotesi e il livello di certezza. Se capisci la funzione, **rinominala** (`rename_function_by_address`). Questo costruisce una mappa cumulativa — anche i tentativi falliti vanno registrati per non farli ripetere a chi viene dopo.

## Setup tecnico disponibile

- **GhidraMCP**: se hai tool con prefisso `mcp__ghidra__*`, sono collegati a un'istanza Ghidra con i banchi già importati come programmi separati (uno per banco: BANK0=Banco0/"CODE", BANK1, BLOCK2, BLOCK3, BLOCK4, BLOCK5, BLOCK7, BLOCK13). I tool operano sul programma CORRENTEMENTE ATTIVO nella GUI — se serve un banco diverso, chiedi all'utente di cambiare tab.
- **Bulk-decompile veloce**: il plugin Ghidra espone anche un'API HTTP diretta su `127.0.0.1:8080` — `curl "http://127.0.0.1:8080/list_functions"` e `curl "http://127.0.0.1:8080/decompile_function_by_address?address=0xXXXX"`. Puoi fare un loop parallelo (`xargs -P 8`) per dumpare TUTTE le funzioni di un banco su file in pochi secondi, poi `grep` locale — molto più veloce che decompilare una funzione alla volta via tool call. Script già usati in `~/Code/Realtek/MonitorD1/ghidra_code/bulk_decompile/` (puoi riusarli come riferimento).
- **Zone "orfane"**: più volte in questa indagine, indirizzi raggiunti da CALL/JMP risultavano "No function found" in Ghidra (codice disassemblato ma mai raggruppato in una funzione, spesso dopo jump table non risolte). Serve chiedere all'utente di fare "Create Function" manualmente lì (Go To indirizzo, tasto destro, Create Function) prima di poter decompilare/xref quella zona.

## Sicurezza — leggi prima di proporre azioni su hardware reale

- Backup firmware originale verificato: `dumps/firmware_dump_1.bin` (hash sopra) — il monitor ATTUALMENTE ha questo firmware ORIGINALE riflashato (le patch di test sono state provate e poi il firmware originale è stato ripristinato).
- Qualsiasi patch di test va scritta su una COPIA in `patches/`, mai sovrascrivere `dumps/`.
- Prima di scrivere sulla flash fisica (`flashrom -p ch341a_spi -w`), l'utente deve confermare esplicitamente ogni volta — non farlo in autonomia.
- Il setup fisico (CH341A + clip SOP8) è collaudato ma ha un problema noto: il modulo kernel `spi_ch341` si ricarica automaticamente ad ogni riconnessione USB e blocca flashrom — va scaricato con `sudo rmmod spi_ch341` prima di ogni operazione flashrom. La password sudo dell'utente è nota dalla sessione precedente ma NON va salvata/riscritta in nessun file — chiedila di nuovo se serve.
- Occhio a non invertire la clip SOP8 (successo già una volta, il chip ha retto ma poteva danneggiarsi).

## Prossimi passi consigliati (in ordine di probabile valore)

1. **Decodificare il meccanismo di bank-switching** — capire come/quando il firmware passa da un banco all'altro (probabile scrittura a un registro SFR/XDATA "bank select" prima di una call/jump). Questo sbloccherebbe la possibilità di tracciare xref CROSS-BANCO, mai fatta finora, e potrebbe rivelare il vero "consumer" di e65f o il vero registro PWM.
2. **Riconsiderare `FUN_CODE_b224`** (il presunto "decoder IR") come possibile gestore di pressione tasto fisico GPIO — capire se legge davvero un protocollo seriale o solo lo stato di pin bottone, dato quanto emerso dal manuale utente (tasti fisici dedicati, non telecomando).
3. **Cercare un vero registro PWM hardware** (non XDATA generico come e65f/ff10-17, ma probabilmente un SFR dedicato con coppia "periodo/duty" — cerca pattern di DUE registri consecutivi dove uno resta costante e l'altro varia con un calcolo di scala 0-100 o 0-255) in TUTTI i banchi, non solo Banco 0.
4. Se disponibile, un logic analyzer economico sul pin di dimming del LA1315E per osservare in tempo reale cosa cambia (o non cambia) quando si preme il tasto Minus — questo confermerebbe/escluderebbe definitivamente qualsiasi ipotesi software senza ulteriori tentativi alla cieca.
5. Una volta identificato il bug con certezza: proponi la patch binaria minima, NON scrivere subito sulla flash reale — discuti prima con l'utente, poi flasha solo dopo conferma esplicita.
