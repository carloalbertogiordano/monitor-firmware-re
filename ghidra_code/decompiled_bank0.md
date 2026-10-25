# Codice decompilato — Banco 0 (offset file 0x000000-0x00ffff)

Estratto da Ghidra (8051:BE:16:default) durante la sessione di analisi. Nomi già assegnati dove capito, `FUN_CODE_xxxx` dove non ancora identificato. Vedi `AI_HANDOFF_PROMPT.md` per il contesto completo e cosa NON seguire (dead-end già investigati).

---

## startup_init_and_call_main (CODE:26eb)
Target del vettore di reset (CODE:0000 -> LJMP 0x26EB).

```c
void startup_init_and_call_main(void)

{
  undefined1 *puVar1;
  char cVar2;
  byte bVar3;
  char cVar4;
  byte *pbVar5;
  byte bVar6;
  byte bVar7;
  undefined1 *puVar8;
  byte *pbVar9;
  byte *pbVar10;

  puVar1 = (undefined1 *)0xff;
  do {
    *puVar1 = 0;
    puVar1 = puVar1 + -1;
  } while (puVar1 != (undefined1 *)0x0);
  puVar8 = &DAT_EXTMEM_e000;
  cVar4 = '\0';
  cVar2 = '\x10';
  do {
    do {
      *puVar8 = 0;
      puVar8 = puVar8 + 1;
      cVar4 = cVar4 + -1;
    } while (cVar4 != '\0');
    cVar2 = cVar2 + -1;
  } while (cVar2 != '\0');
  SP = 0x71;
  pbVar9 = &DAT_CODE_1d92;
  while( true ) {
    bVar3 = 1;
    bVar6 = *pbVar9;
    if (bVar6 == 0) break;
    bVar7 = bVar6 & 0x3f;
    pbVar10 = pbVar9 + 1;
    if (bVar7 >> 5 != 0) {
      bVar3 = bVar6 & 0x1f;
      bVar7 = pbVar9[1];
      pbVar10 = pbVar9 + 2;
      if (bVar7 != 0) {
        bVar3 = bVar3 + 1;
      }
    }
    pbVar9 = pbVar10;
    cVar2 = CARRY1(bVar6 & 0xc0,bVar6 & 0xc0) << 7;
    if ((bVar6 & 0x40) == 0) {
      pbVar5 = (byte *)*pbVar9;
      pbVar9 = pbVar9 + 1;
      do {
        bVar3 = *pbVar9;
        pbVar9 = pbVar9 + 1;
        if (cVar2 < '\0') {
          *(byte *)ZEXT12(pbVar5) = bVar3;
        }
        else {
          *pbVar5 = bVar3;
        }
        pbVar5 = pbVar5 + '\x01';
        bVar7 = bVar7 - 1;
      } while (bVar7 != 0);
    }
    else if (cVar2 < '\0') {
      do {
        bVar3 = *pbVar9;
        pbVar9 = pbVar9 + 1;
        pbVar5 = (byte *)((bVar3 & 0x7f) >> 3 | 0x20);
        bVar6 = *(byte *)((ushort)((bVar3 & 7) + 0xc) + 0x272c);
        if ((char)bVar3 < '\0') {
          bVar6 = bVar6 | *pbVar5;
        }
        else {
          bVar6 = ~bVar6 & *pbVar5;
        }
        *pbVar5 = bVar6;
        bVar7 = bVar7 - 1;
      } while (bVar7 != 0);
    }
    else {
      pbVar10 = *(byte **)pbVar9;
      pbVar9 = pbVar9 + 2;
      do {
        do {
          bVar6 = *pbVar9;
          pbVar9 = pbVar9 + 1;
          *pbVar10 = bVar6;
          pbVar10 = pbVar10 + 1;
          bVar7 = bVar7 - 1;
        } while (bVar7 != 0);
        bVar3 = bVar3 - 1;
      } while (bVar3 != 0);
    }
  }
  FUN_CODE_1183();
  return;
}
```

---

## main_candidate (CODE:1183)
```c
void main_candidate(void)

{
  FUN_CODE_03f7();
  return;
}
```

---

## state_dispatch_arg22_case0 (CODE:03f7)
```c
void state_dispatch_arg22_case0(void)

{
  FUN_CODE_2200(0x22);
  return;
}
```

## state_setter_state00 (CODE:2200)
Parte 1/16 della famiglia di trampolini "state setter" (0x2200, 0x2210 ... 0x22f0). DEAD END: chiamata da centinaia di punti nel codice, troppo ubiqua per essere specifica al display — probabile infrastruttura generica (vedi handoff prompt).
```c
undefined1 FUN_CODE_2200(undefined1 param_1)

{
  DAT_INTMEM_59 = 0;
  uEXTMEMffff = 0;
  return param_1;
}
```

---

## int0_isr_dispatcher (CODE:218f)
Target del vettore Interrupt Esterno 0 (CODE:0003 -> LJMP 0x218F).
```c
undefined1 int0_isr_dispatcher(undefined1 param_1,char param_2)

{
  DAT_EXTMEM_e6dd = DAT_EXTMEM_fff4;
  int0_call_state_dispatch();
  if (param_2 != '\x01') {
    FUN_CODE_055f();
    FUN_CODE_0565();
    if ((DAT_EXTMEM_e6f2 >> 4 == 3) || (DAT_EXTMEM_e6f2 >> 4 == 1)) {
      int0_branch_e6f2_hi_nibble_3or1();
    }
    else {
      int0_branch_e6f2_default();
    }
    FUN_CODE_0577();
  }
  DAT_EXTMEM_fff4 = DAT_EXTMEM_e6dd;
  return param_1;
}
```

## int0_call_state_dispatch (CODE:0559)
```c
void int0_call_state_dispatch(void)

{
  state_dispatch_arg22_case0();
  return;
}
```

## int0_branch_e6f2_hi_nibble_3or1 (CODE:0571)
```c
void int0_branch_e6f2_hi_nibble_3or1(void)

{
  state_dispatch_arg22_case1();
  return;
}
```

## int0_branch_e6f2_default (CODE:056b)
```c
void int0_branch_e6f2_default(void)

{
  state_dispatch_arg22_case1();
  return;
}
```
Nota: entrambi i rami sopra chiamano la STESSA funzione -> nessuna differenza comportamentale osservata.

## state_dispatch_arg22_case1 (CODE:0409)
```c
void state_dispatch_arg22_case1(void)

{
  FUN_CODE_2210(0x22);
  return;
}
```

## state_setter_state01 (CODE:2210)
Parte 2/16 della stessa famiglia di FUN_CODE_2200. Stesso DEAD END.
```c
undefined1 FUN_CODE_2210(undefined1 param_1)

{
  DAT_INTMEM_59 = 1;
  uEXTMEMffff = 1;
  return param_1;
}
```

---

## timer0_isr_latch_fff4 (CODE:0046)
Target del vettore Timer0 (CODE:000B -> LJMP 0x0046).
```c
undefined1 timer0_isr_latch_fff4(undefined1 param_1)

{
  DAT_EXTMEM_e6c5 = DAT_EXTMEM_fff4;
  return param_1;
}
```

---

## Resto della famiglia "reader" (non ancora rinominate, stesso pattern trampolino)

Tutte chiamano il corrispondente state_setter con argomento letterale 0x22 -- stesso dead-end della famiglia sopra.

```c
// CODE:041b
void FUN_CODE_041b(void)
{
  FUN_CODE_2220(0x22);  // state_setter #2
  return;
}

// CODE:042d
void FUN_CODE_042d(void)
{
  FUN_CODE_2230(0x22);  // state_setter #3
  return;
}

// CODE:043f
void FUN_CODE_043f(void)
{
  FUN_CODE_2240(0x22);  // state_setter #4
  return;
}

// CODE:0451
void FUN_CODE_0451(void)
{
  FUN_CODE_2250(0x22);  // state_setter #5
  return;
}
```

---

## Pista aperta non confermata — Banco 1, offset locale 0x3b99 / 0x3d56

(Offset file assoluto 0x13b99 / 0x13d56 -- NON ancora importato/analizzato in Ghidra, questo e' hex grezzo estratto con script Python dal dump)

Due occorrenze IDENTICHE di questa sequenza (differiscono solo nel target di salto finale):

```
90 80 0c 90 e3 e4 74 8c 80 05 90 e3 e4 74 88 f0 a3 80 48
90 e3 e3 e0 ff b4 0f 06 a3 74 c4 f0 80 37
ef b4 0e 08 90 e3 e4 74 c8 f0 80 2b
ef 64 0c 60 04
ef b4 0d 08 90 e3 e4 74 cc f0 80 1a
ef c3 94 08 40 0e
ef d3 94 0b 50 08 90 e3 e4 74 d8 f0 80 06
90 e3 e4 74 84 f0
90 e3 e5 74 40 f0
02 [target salto, differisce tra le 2 occorrenze]
```

Interpretazione (if/else chain su registro R7, scrive byte diversi a XDATA:0xE3E4):
- se R7==0x0F -> scrive 0x8C
- se R7==0x0E -> scrive 0xC8  (200 decimale -- l'indizio che ha attirato l'attenzione)
- se R7==0x0D -> scrive 0xCC
- se R7 in range 0x08-0x0B -> scrive 0xD8
- altrimenti (via confronto su E3E3) -> scrive 0xC4 o 0x84
- sempre, alla fine -> scrive 0x40 a XDATA:0xE3E5

Ipotesi corrente: sembra piu' probabile una tabella "indice modalita' video -> byte di configurazione registro" che una percentuale di luminosita' -- ma NON CONFERMATO, da verificare con piu' contesto (chi chiama questa funzione? cosa rappresenta XDATA:0xE3E4?) una volta che il Banco 1 sara' importato in Ghidra.

**AGGIORNAMENTO (sessione successiva)**: pista CONFERMATA ED ESCLUSA dopo import Banco 1. Vedi `decompiled_bank1.md` per dettagli — xref di DAT_EXTMEM_e3e2 mostra ~80 chiamanti diversi, pattern da "flag globale modo video/timing", non da variabile luminosità. Non seguire oltre.

---

## CATENA BOOT COMPLETA — CANDIDATO PRINCIPALE PER IL BUG (sessione 2026-07-04)

Ricostruita seguendo la catena reale del boot fino al main loop. Vedi `AI_HANDOFF_PROMPT.md` sezione "Catena boot ricostruita" per il riassunto — qui il codice C completo.

### main_candidate (CODE:1183) — AGGIORNATO
Il vero primo byte non era solo una call: è `MOV DPTR,#0xf250` (carica l'indirizzo di main_superloop in DPTR) poi `LJMP 0x03f7` (tail-jump nel dispatcher generico, non ritorna mai qui).

```c
// Bytes reali: 90 f2 50 02 03 f7
// MOV DPTR,#0xf250
// LJMP 0x03f7  (state_dispatch_arg22_case0)
```

### main_superloop (CODE:f250) — la VERA main loop del firmware
Rinominato da "signal_acquire_retry_loop" dopo scoperta che il controllo di uscita è sempre vero (vedi FUN_CODE_feda sotto).

```c
void main_superloop(void)

{
  char in_PSW;

  do {
    signal_lock_reinit_handler();
    while( true ) {
      FUN_CODE_feda();
      if (-1 < in_PSW) break;
      FUN_CODE_05c5();
      FUN_CODE_05cb();
      FUN_CODE_05d1();
      FUN_CODE_05d7();
      FUN_CODE_05dd();
      FUN_CODE_05e3();
      FUN_CODE_05e9();
      FUN_CODE_05ef();
      FUN_CODE_05f5();
      thunk_FUN_CODE_0763();
      FUN_CODE_05fb();
      FUN_CODE_0601();
      FUN_CODE_0607();
      FUN_CODE_1f9d();
    }
  } while( true );
}
```

### FUN_CODE_feda (CODE:feda) — il "check" che non controlla mai nulla
Byte grezzi: `d3 22` = `SETB C` (imposta sempre carry=1) poi `RET`. Ghidra decompila come funzione vuota perché è solo un side-effect di flag, non una variabile. Conseguenza: `if (-1 < in_PSW) break` nel main_superloop è SEMPRE FALSO → il break non scatta mai → il loop interno (13 chiamate dead-end #1) gira all'infinito, `signal_lock_reinit_handler` viene chiamato **una sola volta** all'ingresso nella funzione.

### signal_lock_reinit_handler (CODE:f284)
```c
void signal_lock_reinit_handler(void)

{
  boot_time_peripheral_init_ONCE();
  FUN_CODE_e9d7();
  FUN_CODE_060d();
  FUN_CODE_0613();
  FUN_CODE_f7a7();
  FUN_CODE_0619();
  FUN_CODE_061f();
  DAT_EXTMEM_e6f2 = DAT_EXTMEM_e6f2 & 0xf;
  FUN_CODE_0625(0);
  DAT_EXTMEM_e6f2 = DAT_EXTMEM_e6f2 & 0xf0 | 2;
  DAT_EXTMEM_e6e0 = DAT_EXTMEM_e6e0 & 0xe0 | 7;
  return;
}
```
(FUN_CODE_060d/0613/0619/061f/0625 sono tutte chiamate alla famiglia dead-end #1, già escluse)

### boot_time_peripheral_init_ONCE (CODE:da84) — CANDIDATO PRINCIPALE PER IL BUG
```c
void boot_time_peripheral_init_ONCE(void)

{
  byte bVar1;

  DAT_EXTMEM_ffea = DAT_EXTMEM_ffea | 0x47;
  DAT_EXTMEM_ffe9 = DAT_EXTMEM_ffe9 & 0xef;
  FUN_CODE_f21b(0);
  bEXTMEMfffc = bEXTMEMfffc | 0x1f;
  uEXTMEMfffd = 0;
  uEXTMEMfffe = 0;
  FUN_CODE_f1ae(3,0x7f);
  FUN_CODE_fa27(1);
  FUN_CODE_f9ea();
  FUN_CODE_f317(0);
  bVar1 = DAT_SFR_8e;
  DAT_SFR_8e = bVar1 & 0xc0;
  IE = 0;
  TR0 = 0;
  TF0 = 0;
  TR1 = 0;
  TF1 = 0;
  TR2 = 0;
  TF2 = 0;
  TCLK = 0;
  RCLK = 0;
  TMOD = 0x11;
  bVar1 = T2CON;
  T2CON = bVar1 & 0xfe;
  PT0 = 0;
  PT1 = 0;
  PT2 = 1;
  EA = 1;
  FUN_CODE_06e5(0);
  DAT_EXTMEM_ffea = DAT_EXTMEM_ffea | 0x80;
  FUN_CODE_eab7();
  FUN_CODE_fc07();
  DAT_EXTMEM_ff6c = DAT_EXTMEM_ff6c | 0x60;
  DAT_EXTMEM_ff53 = DAT_EXTMEM_ff53 | 0x3f;
  DAT_EXTMEM_ff47 = 0;
  FUN_CODE_e375();
  FUN_CODE_f3cc();
  FUN_CODE_f64d(0);
  DAT_EXTMEM_ff0f = 0;
  DAT_EXTMEM_ff10 = 0xff;   // <-- CANDIDATO: possibile duty/PWM canale 1, settato a MAX una volta sola
  DAT_EXTMEM_ff11 = 0;
  DAT_EXTMEM_ff12 = 0xff;   // <-- CANDIDATO: possibile duty/PWM canale 2, settato a MAX una volta sola
  DAT_EXTMEM_ff13 = 0;
  DAT_EXTMEM_ff14 = 0xff;   // <-- CANDIDATO: possibile duty/PWM canale 3, settato a MAX una volta sola
  DAT_EXTMEM_ff15 = 0;
  DAT_EXTMEM_ff16 = 0xff;   // <-- CANDIDATO: possibile duty/PWM canale 4, settato a MAX una volta sola
  DAT_EXTMEM_ff17 = 0;
  DAT_EXTMEM_ff02 = DAT_EXTMEM_ff02 & 0x9c | 4;
  DAT_EXTMEM_ff08 = DAT_EXTMEM_ff08 | 0xa0;
  FUN_CODE_0a8d();
  return;
}
```

**Xref confermato**: nessun'altra funzione in Banco 0 scrive mai ff10/ff12/ff14/ff16 dopo questo punto (verificato con `get_xrefs_to` su ognuno dei 4 indirizzi — solo write qui, nessun altro writer, nessun reader statico trovato — coerente con registri hardware write-only tipo PWM).

**NON ANCORA TROVATO**: una funzione "restore/apply impostazioni salvate" che dovrebbe sovrascrivere questi registri col valore corretto (es. luminosità salvata dall'utente) DOPO questo init. Se non esiste in nessuno dei banchi analizzati, è probabilmente la causa diretta del bug.
