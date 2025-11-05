# FIX: Cheapest Blocks Not Updating When Tomorrow Data Available

## Problém
Když se stala dostupná data pro zítřek (po 13:10 CET) a bylo zapnuto `cheapest_blocks_cross_midnight`, senzory pro "cheapest blocks" **neaktualizovaly** svoje hodnoty správně.

## Původní (špatný) kód

```python
for block in config.all_cheapest_blocks():
    if config.cheapest_blocks_cross_midnight and block is not None:
        intervals_for_cheapest = self._today_tomorrow_by_dt  # ← PROBLÉM!
    else:
        intervals_for_cheapest = self._today_day.interval_by_dt
```

**Problém**: `self._today_tomorrow_by_dt` obsahuje **VŠECHNY** intervaly (dnes + zítra) včetně **MINULÝCH**. Když v 14:00 přijdou zítřejší data, algoritmus může najít minulý blok (např. 02:00-05:00) místo budoucího.

## Klíčový princip: Stabilita během dne

### ⚠️ DŮLEŽITÉ: Proč NECHCEME filtrovat minulé intervaly před příchodem zítřejších dat

**ŠPATNÝ přístup** (filtrovat vždy):
```
08:00 bez zítřka → hledá v 08:00-23:59 → najde blok 15:00-18:00
10:00 bez zítřka → hledá v 10:00-23:59 → najde blok 15:00-18:00 (OK)
14:00 bez zítřka → hledá v 14:00-23:59 → najde blok 20:00-23:00 (ZMĚNILO SE!)
16:00 bez zítřka → hledá v 16:00-23:59 → najde blok 21:00-00:00 (ZMĚNILO SE!)
```
→ Hodnota senzoru se **mění každou hodinu** = nepoužitelné! ❌

**SPRÁVNÝ přístup** (stabilní hodnota):
```
08:00 bez zítřka → hledá v 00:00-23:59 → najde blok 02:00-05:00
10:00 bez zítřka → hledá v 00:00-23:59 → najde blok 02:00-05:00 (STEJNÉ!)
14:00 bez zítřka → hledá v 00:00-23:59 → najde blok 02:00-05:00 (STEJNÉ!)
16:00 bez zítřka → hledá v 00:00-23:59 → najde blok 02:00-05:00 (STEJNÉ!)
```
→ Hodnota senzoru je **stabilní celý den** = použitelné v automatizacích! ✓

### Proč to dává smysl?

1. **Senzor ukazuje "nejlevnější blok pro dnešek"**, ne "nejlevnější budoucí blok"
2. **Uživatel kontroluje v automatizaci**, jestli je blok v budoucnosti:
   ```yaml
   - condition: template
     value_template: >
       {{ (state_attr('sensor.cheapest_3_hours', 'start') | as_datetime) > now() }}
   ```
3. **Stabilní hodnota = předvídatelné chování**

## Scénáře

### ✅ Scénář 1: Celý den BEZ zítřejších dat

```
cross_midnight: True
tomorrow_data: NEDOSTUPNÁ ✗

00:00 → hledá v 00:00-23:59 (celý dnešek) → najde 02:00-05:00
08:00 → hledá v 00:00-23:59 (celý dnešek) → najde 02:00-05:00 ← STEJNÉ
14:00 → hledá v 00:00-23:59 (celý dnešek) → najde 02:00-05:00 ← STEJNÉ
20:00 → hledá v 00:00-23:59 (celý dnešek) → najde 02:00-05:00 ← STEJNÉ

✓ Stabilní hodnota celý den
✓ Uživatel ví: "nejlevnější blok pro dnešek byl 02:00-05:00"
✓ Automatizace může zkontrolovat, jestli už proběhl
```

### ✅ Scénář 2: V 14:00 PŘIJDOU zítřejší data (HLAVNÍ FIX!)

```
cross_midnight: True
tomorrow_data: v 14:00 se změní na DOSTUPNÁ ✓

Před 14:00:
  hledá v 00:00-23:59 (celý dnešek) → najde 02:00-05:00

Po 14:00 (přijdou zítřejší data):
  hledá v 14:00-23:59 (zbývající dnes) + 00:00-23:59 (celý zítřek)
  → najde 01:00-04:00 (zítra) ← AKTUALIZACE!

✓ Hodnota se aktualizuje JEDNOU, když přijdou zítřejší data
✓ Nová hodnota ukazuje nejlevnější budoucí blok
✓ Může být buď dnes večer, nebo zítra (podle cen)
```

### ✅ Scénář 3: cross_midnight = False

```
cross_midnight: False

Celý den (bez ohledu na zítřejší data):
  hledá v 00:00-23:59 (celý dnešek) → najde 02:00-05:00

✓ Stabilní hodnota celý den
✓ Nikdy nezahrnuje zítřejší data (respektuje nastavení)
```

### ✅ Scénář 4: Zítřejší data dostupná, ale drahé

```
cross_midnight: True
tomorrow_data: DOSTUPNÁ ✓

Ceny dnes: nízké ráno (02:00-05:00 = 1.5 CZK/kWh)
Ceny zítra: vysoké celý den (nejlevnější 05:00-08:00 = 4.0 CZK/kWh)

V 14:00 (přijdou zítřejší data):
  hledá v 14:00-23:59 (zbývající dnes) + 00:00-23:59 (celý zítřek)
  → najde 21:00-00:00 (dnešní večer) ← Může najít DNEŠNÍ blok!

✓ Algoritmus najde nejlevnější, ať už dnes nebo zítra
✓ V tomto případě dnešní večer je levnější než celý zítřek
```

## Finální implementace

```python
for block in config.all_cheapest_blocks():
    if config.cheapest_blocks_cross_midnight and block is not None:
        # Cross-midnight mode (multi-hour blocks)
        if self._tomorrow_day is not None:
            # MÁME zítřejší data
            # → Filtruj na BUDOUCÍ dnešní + celý zítřek
            intervals_for_cheapest = {
                dt: interval 
                for dt, interval in self._today_day.interval_by_dt.items()
                if interval.dt_local >= self.now
            }
            intervals_for_cheapest.update(self._tomorrow_day.interval_by_dt)
        else:
            # NEMÁME zítřejší data
            # → Celý dnešek (stabilní hodnota)
            intervals_for_cheapest = self._today_day.interval_by_dt.copy()
    else:
        # cross_midnight=False nebo single interval
        # → Vždy celý dnešek (stabilní hodnota)
        intervals_for_cheapest = self._today_day.interval_by_dt.copy()
    
    window = find_cheapest_window(intervals_for_cheapest, ...)
```

## Logická tabulka

| cross_midnight | tomorrow_data | Hledá v intervalech | Stabilita |
|----------------|---------------|---------------------|-----------|
| False | ✗ NE | 00:00-23:59 (dnes) | ✓ Stabilní celý den |
| False | ✓ ANO | 00:00-23:59 (dnes) | ✓ Stabilní celý den |
| True | ✗ NE | 00:00-23:59 (dnes) | ✓ Stabilní celý den |
| True | ✓ ANO | NOW-23:59 (dnes) + 00:00-23:59 (zítra) | ✓ Aktualizuje se jednou v 13:10 |

## Co oprava řeší?

### ✅ PŘED opravou (cross_midnight=True):
```
08:00 bez zítřka → 02:00-05:00 (minulost, ale OK - stabilní)
14:00 se zítřkem → 02:00-05:00 (dnes ráno) ❌ NEAKTUALIZOVALO SE!
                    Mělo najít 01:00-04:00 (zítra)
```

### ✅ PO opravě (cross_midnight=True):
```
08:00 bez zítřka → 02:00-05:00 (stabilní)
14:00 se zítřkem → 01:00-04:00 (zítra) ✓ AKTUALIZOVALO SE!
```

## Testování

### Test 1: Stabilita během dne (před 13:10)
```yaml
# Otestuj ve stejný den v různých časech:
# 08:00
{{ state_attr('sensor.cheapest_3_hours', 'start') }}
# → např. 02:00

# 12:00 (stále stejný výsledek?)
{{ state_attr('sensor.cheapest_3_hours', 'start') }}
# → musí být 02:00 ✓ (STEJNÉ!)
```

### Test 2: Update po příchodu zítřejších dat
```yaml
# Před 13:10:
{{ state_attr('sensor.cheapest_3_hours', 'start') }}
# → např. 02:00 (dnes ráno)

# Po 13:10 (když přijdou zítřejší data):
{{ state_attr('sensor.cheapest_3_hours', 'start') }}
# → např. 01:00 a date je zítra ✓ (ZMĚNILO SE!)
```

### Test 3: Kontrola v automatizaci
```yaml
automation:
  - alias: "Charge battery in cheapest block"
    trigger:
      - platform: time
        at: "{{ state_attr('sensor.cheapest_3_hours', 'start') }}"
    condition:
      # Check if block is in the future
      - condition: template
        value_template: >
          {{ (state_attr('sensor.cheapest_3_hours', 'start') | as_datetime) > now() }}
    action:
      - service: switch.turn_on
        target:
          entity_id: switch.battery_charging
```

## Závěr

**Klíčový princip**: Cheapest blocks jsou **stabilní během dne** a aktualizují se **JEDNOU** když přijdou zítřejší data (v ~13:10).

**Proč to funguje?**
- ✅ Před 13:10: Hledá v celém dnešku (00:00-23:59) → stabilní hodnota
- ✅ Po 13:10: Hledá v budoucích dnešních + zítřek → jedna aktualizace
- ✅ Uživatel kontroluje v automatizaci, jestli je blok v budoucnosti
- ✅ Hodnota se nemění každou hodinu

**Co opravuje?**
- ❌ Původně: `self._today_tomorrow_by_dt` obsahovalo i minulé intervaly
- ✅ Opraveno: Když máme zítřejší data, filtrujeme na `interval.dt_local >= self.now`

**Soubor**: `coordinator.py` (řádky 204-255)  
**Status**: ✅ OPRAVENO - aktualizace při příchodu zítřejších dat
