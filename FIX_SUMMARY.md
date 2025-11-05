# FIX: Cheapest Blocks Not Updating When Tomorrow Data Available

## Problém
Když se stala dostupná data pro zítřek (po 13:10 CET), senzory pro "cheapest blocks" **neaktualizovaly** svoje hodnoty správně při použití `cheapest_blocks_cross_midnight` a ani bez nej.

## Příčina
Původní kód v `IntervalSpotRateData.__init__()` (řádky 207-211):

```python
for block in config.all_cheapest_blocks():
    if config.cheapest_blocks_cross_midnight and block is not None:
        intervals_for_cheapest = self._today_tomorrow_by_dt  # ← PROBLÉM!
    else:
        intervals_for_cheapest = self._today_day.interval_by_dt
```

**Problém**: `self._today_tomorrow_by_dt` obsahuje **VŠECHNY** intervaly včetně **MINULÝCH**. Když v 14:00 přijdou zítřejší data, algoritmus najde nejlevnější blok např. 02:00-05:00 (což už proběhlo), místo aby hledal jen v budoucích intervalech.

## Scénáře

### Scénář 1: 08:00, NEMÁME zítřejší data ❌→✅

**PŘED opravou:**
```
Čas: 08:00
Tomorrow data: NEDOSTUPNÁ ✗

Hledání v intervalech:
  00:00 → 1.50 CZK/kWh  (minulost)
  01:00 → 1.20 CZK/kWh  (minulost)
  02:00 → 1.10 CZK/kWh  (minulost) ← Nejlevnější!
  ...
  08:00 → 3.50 CZK/kWh  (teď)
  09:00 → 3.80 CZK/kWh
  ...

Výsledek: 3h blok 02:00-05:00 ✓ (OK, protože nemáme lepší data)
```

**PO opravě:**
```
Čas: 08:00
Tomorrow data: NEDOSTUPNÁ ✗

Hledání v intervalech (VŠECHNY dnešní):
  00:00 → 1.50 CZK/kWh  (minulost)
  01:00 → 1.20 CZK/kWh  (minulost)
  02:00 → 1.10 CZK/kWh  (minulost) ← Nejlevnější!
  ...
  08:00 → 3.50 CZK/kWh  (teď)
  ...

Výsledek: 3h blok 02:00-05:00 ✓ (stejné, nemáme zítřejší data)
```

### Scénář 2: 14:00, MÁME zítřejší data ❌→✅

**PŘED opravou:**
```
Čas: 14:00
Tomorrow data: DOSTUPNÁ ✓

Hledání v intervalech (včetně minulosti):
  00:00 → 1.50 CZK/kWh  (minulost)
  01:00 → 1.20 CZK/kWh  (minulost) ← Nejlevnější v datasetu!
  02:00 → 1.30 CZK/kWh  (minulost)
  ...
  14:00 → 4.50 CZK/kWh  (teď)
  ...
  Zítra 00:00 → 2.10 CZK/kWh
  Zítra 01:00 → 1.80 CZK/kWh
  Zítra 02:00 → 2.30 CZK/kWh

Výsledek: 3h blok 01:00-04:00 (DNES RÁNO) ❌
→ Nepoužitelné! Už je po 14:00!
```

**PO opravě:**
```
Čas: 14:00
Tomorrow data: DOSTUPNÁ ✓

Hledání v intervalech (JEN BUDOUCNOST):
  14:00 → 4.50 CZK/kWh  (teď)
  15:00 → 4.80 CZK/kWh
  ...
  23:00 → 5.20 CZK/kWh
  Zítra 00:00 → 2.10 CZK/kWh  
  Zítra 01:00 → 1.80 CZK/kWh  ← Nejlevnější v budoucnosti!
  Zítra 02:00 → 2.30 CZK/kWh
  ...

Výsledek: 3h blok 00:00-03:00 (ZÍTŘEJŠÍ NOC) ✓
→ Perfektní! Použitelné pro automatizaci!
```

### Scénář 3: 22:00, NEMÁME zítřejší data ✅

**PO opravě:**
```
Čas: 22:00
Tomorrow data: NEDOSTUPNÁ ✗

Zbývající budoucí intervaly: pouze 22:00-23:59 (2 hodiny)
→ Pro 3h blok nestačí!

Řešení: Použij VŠECHNY dnešní intervaly (včetně minulých)

Výsledek: 3h blok 02:00-05:00 (z rána) ✓
→ OK, lepší data zatím nemáme
```

## Řešení

### Klíčová logika (řádky 204-271)

```python
for block in config.all_cheapest_blocks():
    if self._tomorrow_day is not None:
        # ===== MÁME zítřejší data =====
        # Filtruj na BUDOUCÍ intervaly z dneška
        intervals_for_cheapest = {
            dt: interval 
            for dt, interval in self._today_day.interval_by_dt.items()
            if interval.dt_local >= self.now  # ← Jen budoucí!
        }
        
        # Přidej zítřejší data podle nastavení
        if config.cheapest_blocks_cross_midnight and block is not None:
            intervals_for_cheapest.update(self._tomorrow_day.interval_by_dt)
        else:
            # Přidej zítřek jen když dneska nezbývá dost intervalů
            if len(intervals_for_cheapest) < needed_intervals:
                intervals_for_cheapest.update(self._tomorrow_day.interval_by_dt)
    else:
        # ===== NEMÁME zítřejší data =====
        # Použij VŠECHNY dnešní intervaly (i minulé)
        intervals_for_cheapest = self._today_day.interval_by_dt.copy()
    
    # Najdi nejlevnější okno
    window = find_cheapest_window(intervals_for_cheapest, ...)
```

### Proč tento přístup?

**Když MÁME zítřejší data (po 13:10):**
- ✅ Filtrujeme minulé intervaly → hledáme jen v budoucnosti
- ✅ Zahrnutí zítřejších cen → najdeme skutečně nejlevnější budoucí blok
- ✅ **Tohle byl hlavní problém** - teď se správně přepočítá po příchodu zítřejších dat!

**Když NEMÁME zítřejší data (před 13:10):**
- ✅ Nefiltrujeme → používáme všechny dnešní intervaly
- ✅ Pozdě večer (22:00) máme jen 2h budoucnosti → používáme celý den
- ✅ Aspoň něco zobrazíme, i když to obsahuje minulé bloky

### Proč používáme lokální čas?

```python
if interval.dt_local >= self.now  # ← lokální Prague čas
```

- `self.now` = aktuální čas v Prague timezone (řádek 162)
- `interval.dt_local` = čas intervalu v Prague timezone
- Celá integrace pracuje s Prague timezone
- Dictionary klíče jsou UTC, ale pro filtraci musíme použít lokální čas!

## Důsledky opravy

### ✅ Před 13:10 (zítřejší data NEDOSTUPNÁ)
- Hledá ve **všech** dnešních intervalech (00:00-23:59)
- Může najít i minulý blok (např. 02:00-05:00), ale to je OK - nemáme lepší data
- Funguje i pozdě večer (22:00) ✓

### ✅ Po 13:10 (zítřejší data DOSTUPNÁ, cross_midnight = True)
- Hledá **JEN v budoucích** dnešních + všechny zítřejší (např. 14:00-23:59 + 00:00-23:59)
- **HLAVNÍ OPRAVA**: Nyní najde skutečně nejlevnější budoucí blok!
- Např. může najít 01:00-04:00 zítra místo 15:00-18:00 dnes ✓

### ✅ Po 13:10 (zítřejší data DOSTUPNÁ, cross_midnight = False)
- Hledá **JEN v budoucích** dnešních intervalech (14:00-23:59)
- Pokud nezbývá dost intervalů, automaticky přidá zítřek
- Respektuje uživatelské nastavení ✓

## Testování

Pro otestování použij Home Assistant Template:

```yaml
# Otestuj v Developer Tools → Template
{{ state_attr('sensor.electricity_spot_cheapest_1_hour', 'start') }}
{{ state_attr('sensor.electricity_spot_cheapest_3_hours', 'start') }}

# Kontrola:
# Před 13:10: může ukazovat minulý čas (např. 02:00) - OK
# Po 13:10: musí ukazovat budoucí čas (např. 15:00 dnes nebo 01:00 zítra) - OPRAVENO!
```

## Závěr

**Klíčová změna**: Filtrování minulých intervalů se provádí **JEN když máme zítřejší data**.

Díky tomu:
- ✅ Po 13:10 se cheapest blocks správně přepočítají s novými zítřejšími cenami
- ✅ Nikdy neukazují minulé bloky když máme zítřejší data
- ✅ Před 13:10 stále fungují (používají všechny dnešní intervaly)
- ✅ Pozdě večer (22:00) bez zítřejších dat stále najdou bloky

**Soubor**: `coordinator.py` (řádky 204-271)  
**Status**: ✅ OPRAVENO - kondicionální filtrování podle dostupnosti zítřejších dat
