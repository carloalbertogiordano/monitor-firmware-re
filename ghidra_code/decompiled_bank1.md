# Codice decompilato — Banco 1 (offset file 0x010000-0x01ffff)

Estratto da Ghidra durante sessione di analisi 2026-07-04. Vedi `AI_HANDOFF_PROMPT.md` per contesto e cosa NON seguire.

---

## Pista "e3e2/e3e3/e3e4" — INVESTIGATA A FONDO ED ESCLUSA per luminosità

### videomode_e3e3_to_e3e4_lut_writer (CODE:3b84)
Unico chiamante: CODE:3aee (unconditional call) — un solo punto di chiamata, coerente con setup per UN modo video specifico.

```c
void videomode_e3e3_to_e3e4_lut_writer(void)

{
  if (DAT_EXTMEM_e3e3 == 0xf) {
    DAT_EXTMEM_e3e4 = 0xc4;
  }
  else if (DAT_EXTMEM_e3e3 == 0xe) {
    DAT_EXTMEM_e3e4 = 200;
  }
  else if ((DAT_EXTMEM_e3e3 == 0xc) || (DAT_EXTMEM_e3e3 == 0xd)) {
    DAT_EXTMEM_e3e4 = 0xcc;
  }
  else if (((DAT_EXTMEM_e3e3 < 8) << 7 < '\0') || (0xb < DAT_EXTMEM_e3e3)) {
    DAT_EXTMEM_e3e4 = 0x84;
  }
  else {
    DAT_EXTMEM_e3e4 = 0xd8;
  }
  DAT_EXTMEM_e3e5 = 0x40;
  _0_5 = 0;
  bus_write_variant_a_ff64_65(0x40,DAT_EXTMEM_e3e4);
  DAT_EXTMEM_ff6d = 2;
  DAT_EXTMEM_ff61 = 0xff;
  return;
}
```

### videomode_e3e2_dispatch_or_bus_write (CODE:3bdb)
```c
void videomode_e3e2_dispatch_or_bus_write(void)

{
  undefined1 *puVar1;

  puVar1 = &DAT_EXTMEM_e3e2;
  if (0x3f < DAT_EXTMEM_e3e2) {
    set_e3e4_to_const2();
    puVar1[1] = 0;
    _0_5 = 0;
    bus_write_variant_b_ff60_61(DAT_EXTMEM_e3e5,DAT_EXTMEM_e3e4);
    DAT_EXTMEM_ff6d = 2;
    DAT_EXTMEM_ff61 = 0xff;
    return;
  }
  /* WARNING: Could not recover jumptable at 0x3bed. Too many branches */
  /* WARNING: Treating indirect jump as call */
  (*(code *)((ushort)(DAT_EXTMEM_e3e2 * '\x03') + 0x3bee))();
  return;
}
```
Jump table a 64 rami indicizzata da e3e2*3 — MAI risolta da Ghidra (troppi branch). Se serve capire i singoli "modi", va fatto a mano.

### set_e3e4_to_const2 (CODE:8ba0)
```c
void set_e3e4_to_const2(void)
{
  DAT_EXTMEM_e3e4 = 2;
  return;
}
```

### bus_write_variant_a_ff64_65 (CODE:e090) e bus_write_variant_b_ff60_61 (CODE:dd49)
Quasi identiche, target registri diversi (ff64/ff65/ff70/ff90 vs ff60/ff61/ff68/ff88). Scrivono su un bus interno (probabile I2C/seriale verso periferica) usando e3ec/e3ed/e3ee come address+mask.

```c
void bus_write_variant_a_ff64_65(void)

{
  byte bVar1; byte bVar2; byte bVar3; undefined1 uVar4; byte bVar5; undefined1 *puVar6;

  puVar6 = &DAT_EXTMEM_e3ec;
  uVar4 = FUN_CODE_8d27();
  puVar6[1] = uVar4;
  bVar2 = 5;
  FUN_CODE_f826();
  bVar3 = 0x35;
  DAT_EXTMEM_e3ee = bVar2;
  FUN_CODE_f826();
  bVar1 = DAT_EXTMEM_e700;
  bVar5 = FUN_CODE_8bab();
  puVar6 = &DAT_EXTMEM_e3ec;
  bVar2 = DAT_EXTMEM_e3ec;
  if ((DAT_EXTMEM_e3ec & bVar5) == BANK0_R4) {
    FUN_CODE_8c4b(DAT_EXTMEM_e3ee & bVar5);
    puVar6 = &DAT_EXTMEM_e3ed;
    if ((DAT_EXTMEM_e3ed & bVar1) == bVar3) {
      return;
    }
  }
  FUN_CODE_8c41();
  if (_0_5 == '\0') { *puVar6 = 0x70; } else { *puVar6 = 0x90; }
  DAT_EXTMEM_ff61 = 1;
  DAT_EXTMEM_ff64 = bVar2;
  FUN_CODE_8bd2(DAT_EXTMEM_e3ed,0xff65);
  FUN_CODE_8c05();
  return;
}
```

### PERCHÉ ESCLUSO — evidenza xref decisiva
`get_xrefs_to EXTMEM:e3e2` restituisce **~80 chiamanti diversi** sparsi in un range enorme (CODE:0x39xx-0x66xx). Un valore letto da così tante funzioni sequenziali diverse è coerente con un flag globale "modo video/timing corrente" controllato da decine di handler per-modo, NON con una variabile di luminosità (che avrebbe un numero di lettori molto più piccolo e concentrato). **Conclusione: pista chiusa, non seguire oltre per il bug luminosità.**

---

## Pista OSD bar-graph renderer — ESCLUSA (clamp corretto)

### osd_bargraph_render_percent_to_position (CODE:c07a)
Contiene il clamp `if (100 < param_3) DAT_EXTMEM_e37d = 100;` — **funziona correttamente**. Il resto è matematica a virgola fissa per convertire percentuale 0-100 in coordinate X/Y per disegnare uno slider/gauge OSD. Chiamata da ~10 punti (90c2, ad76, c7df, 6bc2, 6be2, 6c5d, 6d7a, 6c79, 6c97, 6cba) — riusata per TUTTI gli slider del menu (luminosità/contrasto/volume/ecc), quindi è codice di rendering condiviso, non hardware reale.

```c
byte osd_bargraph_render_percent_to_position(char param_1,char param_2,byte param_3)

{
  bool bVar1; undefined1 uVar2; char cVar3; char cVar4; undefined1 uVar5; byte bVar6; char *pcVar7;

  FUN_CODE_a558(0xe37e);
  DAT_EXTMEM_e37d = param_3;
  FUN_CODE_1724(0xe384);
  nop(); nop(); nop(); nop();
  if (100 < param_3) {
    DAT_EXTMEM_e37d = 100;
  }
  if ((DAT_EXTMEM_e37d < 0x33) << 7 < '\0') {
    cVar4 = (DAT_EXTMEM_e37d < 0x32) << 7;
    if (DAT_EXTMEM_e37d >= 0x32) {
      return DAT_EXTMEM_e383;
    }
    FUN_CODE_a508(DAT_EXTMEM_e37d - 0x32,0xe380,DAT_EXTMEM_e37d);
    bVar6 = param_2 - (cVar4 >> 7);
    cVar3 = DAT_EXTMEM_e383 - bVar6;
    cVar4 = DAT_EXTMEM_e382 - (param_1 - (((DAT_EXTMEM_e383 < bVar6) << 7) >> 7));
  }
  else {
    cVar4 = '\0';
    FUN_CODE_a507(0xe382,DAT_EXTMEM_e37d - 0x32);
    bVar6 = param_2 - (cVar4 >> 7);
    cVar3 = DAT_EXTMEM_e37f - bVar6;
    cVar4 = DAT_EXTMEM_e37e - (param_1 - (((DAT_EXTMEM_e37f < bVar6) << 7) >> 7));
  }
  FUN_CODE_17ba(cVar4,cVar3);
  FUN_CODE_14a4(0,0,0,10);
  FUN_CODE_a538(0x32);
  FUN_CODE_1718(0xe384);
  uVar5 = FUN_CODE_a4a9();
  FUN_CODE_152f(uVar5);
  cVar3 = '\0';
  uVar5 = BANK0_R0; uVar2 = BANK0_R1; cVar4 = BANK0_R2;
  FUN_CODE_15d7(0,0,0,5,BANK0_R3);
  if (cVar3 < '\0') {
    bVar6 = FUN_CODE_a4af();
    cVar4 = cVar4 - (((0xf5 < bVar6) << 7) >> 7);
    FUN_CODE_1718(0xe384,uVar5,uVar2,cVar4,bVar6 + 10);
  }
  cVar3 = (DAT_EXTMEM_e37d < 0x33) << 7;
  bVar6 = FUN_CODE_a4a9(DAT_EXTMEM_e37d - 0x33);
  if (cVar3 < '\0') {
    FUN_CODE_152f(bVar6);
    bVar1 = CARRY1(DAT_EXTMEM_e381,bVar6);
    pcVar7 = &DAT_EXTMEM_e380;
  }
  else {
    FUN_CODE_152f(bVar6);
    bVar1 = CARRY1(DAT_EXTMEM_e383,bVar6);
    pcVar7 = &DAT_EXTMEM_e382;
  }
  return *pcVar7 + (cVar4 - ((bVar1 << 7) >> 7));
}
```

### osd_bargraph_setup_wrapper (CODE:c7df)
NOTA: `bVar4 = 100 - DAT_EXTMEM_e332` NON clampa e332 prima — se e332>100 c'è underflow byte. Ma e332 è variabile scratch condivisa da moltissime funzioni (xref: c7df, ad76, ec30, ba12, 846a, e8cc, efc4, a4db, f021, a176, cddf, cdeb, a88a, c240, dccc + scritture dirette a 80f8/a241/a1cd) — troppo generica per essere specifica alla luminosità.

```c
void osd_bargraph_setup_wrapper(byte param_1)

{
  bool bVar1; undefined1 uVar2; undefined1 uVar3; byte bVar4; byte bVar5; char cVar6;

  uVar3 = FUN_CODE_a457(0xe332);
  FUN_CODE_e7b2();
  uVar2 = 1;
  DAT_EXTMEM_e335 = param_1;
  DAT_EXTMEM_e336 = uVar3;
  FUN_CODE_e7b2();
  uVar3 = BANK0_R7;
  cVar6 = (100 < DAT_EXTMEM_e332) << 7;
  bVar4 = 100 - DAT_EXTMEM_e332;
  DAT_EXTMEM_e337 = param_1;
  DAT_EXTMEM_e338 = uVar2;
  FUN_CODE_a4b2(0xe335,param_1,uVar2);
  FUN_CODE_a577(0xe335);
  FUN_CODE_a4d4();
  FUN_CODE_a522();
  BANK0_R7 = uVar3;
  osd_bargraph_render_percent_to_position();
  DAT_EXTMEM_e333 = param_1;
  DAT_EXTMEM_e334 = bVar4;
  FUN_CODE_0a75();
  if (cVar6 < '\0') {
    return;
  }
  _0_1 = 0;
  FUN_CODE_0a0f();
  FUN_CODE_0a87();
  DAT_EXTMEM_e339 = FUN_CODE_a5ae(0xe333);
  DAT_EXTMEM_e33a = bVar4;
  FUN_CODE_0ae7();
  if (bVar4 != 1) {
    FUN_CODE_0ae7();
    bVar1 = bVar4 < 3;
    if (bVar4 != 3) goto LAB_CODE_c86f;
  }
  FUN_CODE_0af9();
  bVar4 = DAT_EXTMEM_e33a - bVar4 & 1;
  bVar1 = CARRY1(DAT_EXTMEM_e33a,bVar4);
  DAT_EXTMEM_e33a = DAT_EXTMEM_e33a + bVar4;
  bVar5 = -((bVar1 << 7) >> 7);
  bVar1 = CARRY1(DAT_EXTMEM_e339,bVar5);
  DAT_EXTMEM_e339 = DAT_EXTMEM_e339 + bVar5;
LAB_CODE_c86f:
  cVar6 = bVar1 << 7;
  bVar5 = FUN_CODE_a4f7(0xe339);
  if ((param_1 ^ 0x80) < 0x80U - (((bVar5 < 1U - (cVar6 >> 7)) << 7) >> 7)) {
    param_1 = 0;
    bVar4 = 1;
  }
  DAT_EXTMEM_e339 = param_1;
  DAT_EXTMEM_e33a = bVar4;
  FUN_CODE_0a45();
  FUN_CODE_08c5(2);
  _0_1 = 1;
  FUN_CODE_0a0f();
  return;
}
```

---

## Fatto critico che riorienta la ricerca (test hardware 2026-07-04)

Bug si manifesta anche con SOLO HDMI collegato (niente USB-C) → bug puramente interno al firmware, scatenato dal lock/rilevamento segnale video. Cercare codice legato a interrupt di segnale video (candidato: `int0_isr_dispatcher` in Banco 0, CODE:218f), NON più codice menu OSD (ampiamente escluso sopra).
