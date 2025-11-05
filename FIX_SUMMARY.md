# FIX: Cheapest Blocks Not Updating When Tomorrow Data Available

## Problém
Když se stala dostupná data pro zítřek (po 13:10 CET), senzory pro "cheapest blocks" **neaktualizovaly** svoje hodnoty a **stále ukazovaly bloky z včerejška nebo z rána**, i když už byla k dispozici nová data se zítřejšími cenami.

## Příčina
Původní kód v `IntervalSpotRateData.__init__()` (řádky 207-211) používal:

```python
for block in config.all_cheapest_blocks():
    if config.cheapest_blocks_cross_midnight and block is not None:
        intervals_for_cheapest = self._today_tomorrow_by_dt  # ← CHYBA!
    else:
        intervals_for_cheapest = self._today_day.interval_by_dt  # ← CHYBA!
```

**Problém č. 1**: `self._today_tomorrow_by_dt` obsahuje **VŠECHNY** intervaly z dneška a zítřka, včetně **MINULÝCH** intervalů (které už proběhly).

**Problém č. 2**: Algoritmus `find_cheapest_window()` pak hledá nejlevnější okno v celém datasetu, včetně minulých hodin.

**Výsledek**: Senzor ukazuje např. "nejlevnější blok 02:00-05:00", i když je aktuálně 14:00! To je naprosto nepoužitelné pro automatizace.

## Příklad scenáře

### Původní chování (ŠPATNĚ) ❌
```
Čas: 14:00 Prague
Tomorrow data: Dostupná ✓

Hledání v intervalech:
  00:00 → 1.50 CZK/kWh  ← MINULOST! 
  01:00 → 1.20 CZK/kWh  ← MINULOST!
  02:00 → 1.10 CZK/kWh  ← MINULOST!
  ...
  14:00 → 4.50 CZK/kWh  ← TEĎ
  15:00 → 4.80 CZK/kWh  
  ...
  23:00 → 5.20 CZK/kWh
  Zítra 00:00 → 2.10 CZK/kWh
  Zítra 01:00 → 1.80 CZK/kWh
  Zítra 02:00 → 2.30 CZK/kWh

Výsledek: Nejlevnější 3h blok = 01:00-04:00 (dnes ráno) ❌
→ Úplně k ničemu, protože už je po 14:00!
```

### Opravené chování (SPRÁVNĚ) ✓
```
Čas: 14:00 Prague
Tomorrow data: Dostupná ✓

Hledání v intervalech (JEN BUDOUCNOST):
  14:00 → 4.50 CZK/kWh  ← TEĎ (začátek)
  15:00 → 4.80 CZK/kWh  
  ...
  23:00 → 5.20 CZK/kWh
  Zítra 00:00 → 2.10 CZK/kWh  ← Levné!
  Zítra 01:00 → 1.80 CZK/kWh  ← Velmi levné!
  Zítra 02:00 → 2.30 CZK/kWh  ← Levné!
  ...

Výsledek: Nejlevnější 3h blok = 00:00-03:00 (zítřejší noc) ✓
→ Perfektní! Baterii nabijeme zítra v noci!
```

## Řešení

### Co bylo změněno (řádky 204-259)

Místo původního kódu jsem implementoval **tři-krokový proces**:

**KROK 1**: Filtrovat minulé intervaly (používá LOKÁLNÍ čas!)
```python
# self.now je už v lokálním čase (Prague timezone) - řádek 162
# interval.dt_local je také v lokálním čase (Prague timezone)
# Porovnáváme LOKÁLNÍ časy, NE UTC!

intervals_for_cheapest = {
    dt: interval 
    for dt, interval in self._today_day.interval_by_dt.items()
    if interval.dt_local >= self.now  # ← Porovnává lokální Prague čas!
}
```

**KROK 2**: Přidat zítřejší data (když jsou dostupná)
```python
if self._tomorrow_day is not None:
    if config.cheapest_blocks_cross_midnight and block is not None:
        # Cross-midnight zapnuto → VŽDY přidat zítřejší data
        intervals_for_cheapest.update(self._tomorrow_day.interval_by_dt)
    else:
        # Cross-midnight vypnuto → přidat jen když dneska nezbývá dost intervalů
        if available_today < needed_intervals:
            intervals_for_cheapest.update(self._tomorrow_day.interval_by_dt)
```

**KROK 3**: Validovat a hledat nejlevnější okno
```python
if not intervals_for_cheapest:
    logger.warning("Žádné budoucí intervaly k dispozici")
    continue

window = find_cheapest_window(intervals_for_cheapest, hours=block, interval=config.interval)
```

### ⚠️ Důležité: Použití lokálního času

**Proč používáme `interval.dt_local >= self.now` a NE `dt >= now_utc`?**

Protože celá integrace pracuje v **Prague timezone (Europe/Prague)**:

- `self.now = get_now(config.zoneinfo)` → čas v Prague timezone
- `self.today_date = self.now.date()` → dnešní datum v Prague timezone  
- `interval.dt_local` → čas intervalu v Prague timezone
- `self._today_day` obsahuje intervaly podle **lokálního data** (řádek 193: `if rate_hour.dt_local.date() == self.today_date`)

Kdyby jsme použili UTC časy, filtrace by byla špatně, zvlášť během přechodů na letní/zimní čas!

### Přesné změny v kódu

**MÍSTO TOHOTO:**
```python
for block in config.all_cheapest_blocks():
    if config.cheapest_blocks_cross_midnight and block is not None:
        intervals_for_cheapest = self._today_tomorrow_by_dt
    else:
        intervals_for_cheapest = self._today_day.interval_by_dt

    try:
        window = find_cheapest_window(
            intervals_for_cheapest,
            hours=block,
            interval=config.interval,
        )
```

**POUŽIJ TOTO:**
```python
for block in config.all_cheapest_blocks():
    # STEP 1: Start with today's FUTURE intervals only (filter out past)
    # Use LOCAL time for comparison, not UTC!
    intervals_for_cheapest = {
        dt: interval 
        for dt, interval in self._today_day.interval_by_dt.items()
        if interval.dt_local >= self.now  # Compare LOCAL times!
    }
    
    # STEP 2: Handle tomorrow's data based on cross_midnight setting
    if self._tomorrow_day is not None:
        if config.cheapest_blocks_cross_midnight and block is not None:
            # Cross-midnight enabled: ALWAYS add tomorrow's data
            intervals_for_cheapest.update(self._tomorrow_day.interval_by_dt)
        else:
            # Cross-midnight disabled: only add tomorrow if not enough intervals today
            available_today = len(intervals_for_cheapest)
            needed_intervals = 1 if block is None else (block if config.interval == SpotRateIntervalType.Hour else block * 4)
            
            if available_today < needed_intervals:
                intervals_for_cheapest.update(self._tomorrow_day.interval_by_dt)
    
    # STEP 3: Validate and find cheapest window
    if not intervals_for_cheapest:
        logger.warning("No future intervals available")
        continue

    try:
        window = find_cheapest_window(
            intervals_for_cheapest,
            hours=block,
            interval=config.interval,
        )
```

## Důsledky

### ✅ **Před 13:10** (zítřejší data ještě nedostupná)
- Hledá jen ve zbývajících hodinách dneška (14:00-23:59 Prague čas)
- Funguje správně ✓

### ✅ **Po 13:10** (zítřejší data dostupná, cross_midnight = True)
- Hledá ve zbývajících hodinách dneška + celý zítřek (14:00-23:59 + 00:00-23:59 Prague čas)
- **OPRAVENO**: Nyní najde skutečně nejlevnější blok ✓

### ✅ **Po 13:10** (zítřejší data dostupná, cross_midnight = False)
- Hledá jen ve zbývajících hodinách dneška (14:00-23:59 Prague čas)
- Pokud nezbývá dost intervalů, automaticky přidá zítřejší data
- Respektuje uživatelské nastavení ✓

### ✅ **Pozdě večer** (23:45 Prague čas)
- Zbývá jen 15 minut dneška
- Automaticky přidá zítřejší data (bez ohledu na cross_midnight)
- Najde použitelné bloky ✓

## Závěr

**Tato oprava zajistí, že cheapest blocks senzory budou vždy zobrazovat relevantní budoucí bloky, nikdy ne minulé hodiny.**

Když se objeví nová data pro zítřek (obvykle kolem 13:10 Prague čas), senzory se automaticky přepočítají a začnou zahrnovat zítřejší ceny do hledání nejlevnějších bloků.

**Soubor**: `coordinator.py` (řádky 204-259)  
**Status**: ✅ OPRAVENO  
**Klíčová změna**: Použití `interval.dt_local >= self.now` pro správnou filtraci v lokálním čase (Prague timezone)
