# Pepsi Max-tilbud i Home Assistant

Et dashboard-kort, der viser de **5 billigste Pepsi Max-tilbud pr. liter** fra [tilbudsugen.dk](https://www.tilbudsugen.dk/offer/pepsi-max). Hver række har butikkens logo, butiksnavn, produkt, pakke, pris, literpris og gyldighedsdato. Et tryk på rækken åbner tilbuddet.

Der er ingen filer, ingen YAML i `configuration.yaml` og ingen ekstra integration at installere. Den eneste afhængighed er kortet **Button Card**.

> Testet på Home Assistant 2026.10. Data hentes fra tilbudsugens egen side, så det kan gå i stykker, hvis de ændrer sidens opbygning.

---

## Det skal du bruge

| Krav | Bemærkning |
|---|---|
| Home Assistant 2026.x | Integrationen **Scrape** er indbygget |
| **Button Card** (`custom:button-card`) | Installeres via HACS → Frontend → "Button Card" |
| Internetadgang | Både fra Home Assistant (henter siden) og fra din browser (henter butikslogoerne) |

---

## Sådan sætter du det op

### Trin 1 – Opret Scrape-integrationen

1. Gå til **Indstillinger → Enheder & tjenester → Tilføj integration** og vælg **Scrape**.
2. Udfyld:
   - **Resource:** `https://www.tilbudsugen.dk/offer/pepsi-max`
   - **Method:** GET
   - Under **Additional / Yderligere:** sæt **Headers** til
     ```yaml
     User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36
     ```
     og **Timeout** til `30`.
3. Gem.

### Trin 2 – Opret 5 sensorer

Tryk **Tilføj emne / Add sensor** på Scrape-integrationen. Gør det **fem gange**, med disse værdier:

| Navn | `rang` i skabelonen |
|---|---|
| Pepsi Max billigst 1 | 0 |
| Pepsi Max billigst 2 | 1 |
| Pepsi Max billigst 3 | 2 |
| Pepsi Max billigst 4 | 3 |
| Pepsi Max billigst 5 | 4 |

For hver sensor:
- **Select (CSS-vælger):** `script#__NEXT_DATA__`
- **Value template:** indsæt skabelonen nedenfor, og ret kun tallet i første linje (`{% set rang = 0 %}`).
- Lad alle andre felter stå tomme.

Entiteterne bliver til `sensor.pepsi_max_billigst_1` … `sensor.pepsi_max_billigst_5`. Navnene skal være præcis dem, for kortet læser dem.

**Skabelon (Value template):**

```jinja
{% set rang = 0 %}{# 0 = billigst, 1 = næstbilligst, 2 = ... (skift tallet på hver sensor) #}
{% set ekskluder = ['løvbjerg', 'spar', 'abc'] %}{# butikker du IKKE vil se: små bogstaver, delnavn er nok #}
{% set items = (value | from_json).props.pageProps.data.pageOfOrganicProductOffers['items'] %}
{% set today = now().date().isoformat() %}
{% set ns = namespace(l=[], chains=[], best=[], skip=false) %}
{% for it in items %}
  {% set lp = it.pricePerFullUnitOfQuantityType | float(0) %}
  {% set nm = it.brand.name ~ ', ' ~ it.productName.productName %}
  {% set cl = it.chain.name | lower %}
  {% set ns.skip = false %}
  {% for e in ekskluder %}{% if e in cl %}{% set ns.skip = true %}{% endif %}{% endfor %}
  {% if lp > 0 and it.quantityType == 'ltr.' and not it.isBorderShop and it.endDate >= today and 'pepsi' in (it.brand.name | lower) and 'sodastream' not in (nm | lower) and 'koncentrat' not in (nm | lower) and not ns.skip %}
    {% set ns.l = ns.l + [{'lp': lp + (0 if nm == 'Pepsi Max, Cola' else 0.001), 'rlp': lp, 'it': it, 'nm': nm}] %}
  {% endif %}
{% endfor %}
{% for c in ns.l | sort(attribute='lp') %}
  {% if c.it.chain.name not in ns.chains %}
    {% set ns.chains = ns.chains + [c.it.chain.name] %}
    {% set ns.best = ns.best + [c] %}
  {% endif %}
{% endfor %}
{% if ns.best | length > rang %}
  {% set c = ns.best[rang] %}{% set b = c.it %}
  {{ c.rlp ~ '|' ~ b.chain.name ~ '|' ~ c.nm ~ '|' ~ b.units ~ 'x' ~ (b.quantity | float(0)) ~ '|' ~ b.price ~ '|' ~ b.startDate ~ '|' ~ b.endDate ~ '|' ~ b.id }}
{% else %}Ingen{% endif %}
```

Hver sensors værdi er en tekst på formen:
`literpris|butik|produkt|antalxstørrelse|pris|startdato|slutdato|tilbuds-id`
fx `6.63|Meny|Pepsi Max, Cola|1x1.5|9.95|2026-10-23|2026-10-29|11370615`

Tjek under **Udviklerværktøjer → Tilstande**, at sensorerne har en værdi. Står der `Ingen`, er der ingen tilbud, der opfylder kravene (se Tilpasning).

### Trin 3 – Opdatering to gange i døgnet (anbefalet)

Scrape henter som standard siden hvert 10. minut. Det er unødvendigt ofte.

1. Gå til Scrape-integrationen → de tre prikker → **Systemindstillinger** → slå **"Aktivér automatisk hentning af opdateringer"** fra.
2. Opret en automation (**Indstillinger → Automatiseringer → Ny → Rediger i YAML**) og indsæt:

```yaml
alias: Pepsi Max tilbud - opdater to gange i døgnet
description: Henter tilbudsugen-siden (Scrape) kl. 06 og 18. Opdaterer én sensor, og alle fem sensorer får nye data.
triggers:
  - trigger: time
    at:
      - "06:00:00"
      - "18:00:00"
conditions: []
actions:
  - action: homeassistant.update_entity
    target:
      entity_id: sensor.pepsi_max_billigst_1
mode: single
```

### Trin 4 – Tilføj kortet

1. Åbn dit dashboard → **Rediger** → **Tilføj kort** → **Manuelt**.
2. Slet det, der står, og indsæt YAML'en nedenfor. Gem.

```yaml
type: vertical-stack
cards:
- type: custom:button-card
  show_icon: false
  show_name: false
  show_state: false
  custom_fields:
    left: <div style="font-size:11px;letter-spacing:3px;opacity:.8">TILBUDSJAGT</div><div style="font-size:28px;font-weight:800;letter-spacing:1px;line-height:1.15">PEPSI MAX</div><div style="font-size:12px;opacity:.85;margin-top:2px">Billigste tilbud pr. liter</div>
    right: '<svg width="50" height="82" viewBox="0 0 60 100" xmlns="http://www.w3.org/2000/svg" style="transform: rotate(10deg); filter: drop-shadow(0 4px 6px rgba(0,0,0,0.55));"><defs><linearGradient id="pmBody" x1="0" x2="1"><stop offset="0" stop-color="#05070d"/><stop offset="0.45" stop-color="#2a3246"/><stop offset="1" stop-color="#05070d"/></linearGradient><linearGradient id="pmMetal" x1="0" x2="1"><stop offset="0" stop-color="#8a93a6"/><stop offset="0.5" stop-color="#f1f4fa"/><stop offset="1" stop-color="#8a93a6"/></linearGradient></defs><path d="M8 12 L8 88 Q8 96 30 96 Q52 96 52 88 L52 12 Q52 18 30 18 Q8 18 8 12 Z" fill="url(#pmBody)"/><ellipse cx="30" cy="12" rx="22" ry="5.5" fill="url(#pmMetal)"/><ellipse cx="30" cy="12" rx="16" ry="3.2" fill="#cfd5e2"/><path d="M8 42 Q30 50 52 42 L52 62 Q30 70 8 62 Z" fill="#1976d2"/><text x="30" y="35" font-size="9.5" font-weight="900" fill="#4fc3f7" text-anchor="middle" font-family="Arial Black, Arial, sans-serif">MAX</text><rect x="13" y="22" width="4" height="68" rx="2" fill="#ffffff" opacity="0.13"/></svg>'
  styles:
    card:
    - background: 'linear-gradient(120deg, #0b2a6b 0%, #0d47a1 55%, #d32f2f 140%)'
    - border-radius: 16px
    - padding: 14px 18px
    - color: '#fff'
    - box-shadow: 0 4px 18px rgba(13,71,161,0.45)
    grid:
    - grid-template-areas: '"left right"'
    - grid-template-columns: 1fr auto
    - align-items: center
    custom_fields:
      left:
      - justify-self: start
      - text-align: left
      right:
      - justify-self: end
  tap_action:
    action: none
- type: custom:button-card
  entity: sensor.pepsi_max_billigst_1
  show_icon: false
  show_name: false
  show_state: false
  tap_action:
    action: url
    url_path: '[[[ const p = String(entity.state || '''').split(''|''); return p.length >= 8 ? ''https://www.tilbudsugen.dk/single/'' + p[7] : ''https://www.tilbudsugen.dk/offer/pepsi-max''; ]]]'
  custom_fields:
    rank: '[[[ const p = String(entity.state || '''').split(''|''); const chain = p[1] || ''''; if (!chain) return ''''; const key = chain.toLowerCase(); const map = {''meny'': ''meny.png'', ''rema 1000'': ''rema-1000.svg'', ''kvickly'': ''kvickly.png'', ''netto'': ''netto.png'', ''superbrugsen'': ''super-brugsen.png'', ''365discount'': ''coop365.png'', ''bilka'': ''bilka.png'', ''lidl'': ''lidl.png'', ''løvbjerg'': ''loevbjerg.png'', ''spar'': ''SPAR.jpg'', ''fakta'': ''fakta.png'', ''calle'': ''calle.png''}; const slug = key.replace(/ø/g, ''oe'').replace(/å/g, ''aa'').replace(/æ/g, ''ae'').replace(/[^a-z0-9]+/g, ''-'').replace(/^-+|-+$/g, ''''); const file = map[key] || (slug + ''.png''); const ini = chain.replace(/[^A-Za-zÆØÅæøå0-9 ]/g, '''').split('' '').filter(Boolean).slice(0, 2).map(function (w) { return w[0].toUpperCase(); }).join(''''); const edge = ''border:2px solid #ffd54f;box-shadow:0 0 10px rgba(255,213,79,0.6);''; return ''<div style="width:46px;height:46px;border-radius:12px;background:#fff;position:relative;overflow:hidden;display:flex;align-items:center;justify-content:center;'' + edge + ''"><span style="position:absolute;color:#333;font-weight:800;font-size:15px">'' + ini + ''</span><img src="https://www.tilbudsugen.dk/chains/'' + file + ''" referrerpolicy="no-referrer" style="position:relative;width:100%;height:100%;object-fit:contain;background:#fff;padding:3px;box-sizing:border-box" onerror="this.remove()"></div>''; ]]]'
    info: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''<span style="opacity:.6">Ingen tilbud</span>''; const u = parseInt(p[3].split(''x'')[0]); const sz = String(Number(p[3].split(''x'')[1])).replace(''.'', '',''); const pack = (u > 1 ? u + '' × '' : '''') + sz + '' l''; const later = p[5] > today; return ''<div style="font-weight:700;font-size:16px;line-height:1.2">'' + p[1] + ''</div><div style="opacity:.75;font-size:12px;margin-top:2px">'' + p[2] + '' · '' + pack + '' = '' + fmt(p[4]) + '' kr</div><div style="font-size:11px;margin-top:3px;opacity:.9">'' + dd(p[5]) + ''–'' + dd(p[6]) + (later ? '' <span style="background:#ffb300;color:#111;border-radius:6px;padding:0 6px;margin-left:4px;font-weight:700">starter snart</span>'' : '''') + ''</div>''; ]]]'
    price: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''''; const first = String(((states[''sensor.pepsi_max_billigst_1''] || {}).state) || '''').split(''|'')[0]; const min = Number(first) || Number(p[0]); const pct = Math.max(8, Math.min(100, Math.round(min / Number(p[0]) * 100))); return ''<div style="text-align:right"><span style="font-size:26px;font-weight:800;line-height:1">'' + fmt(p[0]) + ''</span><span style="font-size:12px;opacity:.75"> kr/l</span></div><div style="margin-top:6px;height:4px;width:96px;border-radius:2px;background:rgba(255,255,255,.18);margin-left:auto"><div style="height:4px;width:'' + pct + ''%;border-radius:2px;background:#4fc3f7"></div></div>''; ]]]'
  styles:
    card:
    - padding: 10px 14px
    - border-radius: 14px
    - background: 'linear-gradient(135deg, #0d47a1 0%, #1976d2 100%)'
    - border: 1px solid rgba(255,255,255,0.14)
    - color: '#fff'
    - box-shadow: 0 0 18px rgba(25,118,210,0.55)
    grid:
    - grid-template-areas: '"rank info price"'
    - grid-template-columns: 52px 1fr auto
    - align-items: center
    - column-gap: 12px
    custom_fields:
      rank:
      - justify-self: center
      info:
      - justify-self: start
      - text-align: left
      price:
      - justify-self: end
- type: custom:button-card
  entity: sensor.pepsi_max_billigst_2
  show_icon: false
  show_name: false
  show_state: false
  tap_action:
    action: url
    url_path: '[[[ const p = String(entity.state || '''').split(''|''); return p.length >= 8 ? ''https://www.tilbudsugen.dk/single/'' + p[7] : ''https://www.tilbudsugen.dk/offer/pepsi-max''; ]]]'
  custom_fields:
    rank: '[[[ const p = String(entity.state || '''').split(''|''); const chain = p[1] || ''''; if (!chain) return ''''; const key = chain.toLowerCase(); const map = {''meny'': ''meny.png'', ''rema 1000'': ''rema-1000.svg'', ''kvickly'': ''kvickly.png'', ''netto'': ''netto.png'', ''superbrugsen'': ''super-brugsen.png'', ''365discount'': ''coop365.png'', ''bilka'': ''bilka.png'', ''lidl'': ''lidl.png'', ''løvbjerg'': ''loevbjerg.png'', ''spar'': ''SPAR.jpg'', ''fakta'': ''fakta.png'', ''calle'': ''calle.png''}; const slug = key.replace(/ø/g, ''oe'').replace(/å/g, ''aa'').replace(/æ/g, ''ae'').replace(/[^a-z0-9]+/g, ''-'').replace(/^-+|-+$/g, ''''); const file = map[key] || (slug + ''.png''); const ini = chain.replace(/[^A-Za-zÆØÅæøå0-9 ]/g, '''').split('' '').filter(Boolean).slice(0, 2).map(function (w) { return w[0].toUpperCase(); }).join(''''); const edge = ''border:1px solid rgba(255,255,255,0.25);''; return ''<div style="width:46px;height:46px;border-radius:12px;background:#fff;position:relative;overflow:hidden;display:flex;align-items:center;justify-content:center;'' + edge + ''"><span style="position:absolute;color:#333;font-weight:800;font-size:15px">'' + ini + ''</span><img src="https://www.tilbudsugen.dk/chains/'' + file + ''" referrerpolicy="no-referrer" style="position:relative;width:100%;height:100%;object-fit:contain;background:#fff;padding:3px;box-sizing:border-box" onerror="this.remove()"></div>''; ]]]'
    info: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''<span style="opacity:.6">Ingen tilbud</span>''; const u = parseInt(p[3].split(''x'')[0]); const sz = String(Number(p[3].split(''x'')[1])).replace(''.'', '',''); const pack = (u > 1 ? u + '' × '' : '''') + sz + '' l''; const later = p[5] > today; return ''<div style="font-weight:700;font-size:16px;line-height:1.2">'' + p[1] + ''</div><div style="opacity:.75;font-size:12px;margin-top:2px">'' + p[2] + '' · '' + pack + '' = '' + fmt(p[4]) + '' kr</div><div style="font-size:11px;margin-top:3px;opacity:.9">'' + dd(p[5]) + ''–'' + dd(p[6]) + (later ? '' <span style="background:#ffb300;color:#111;border-radius:6px;padding:0 6px;margin-left:4px;font-weight:700">starter snart</span>'' : '''') + ''</div>''; ]]]'
    price: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''''; const first = String(((states[''sensor.pepsi_max_billigst_1''] || {}).state) || '''').split(''|'')[0]; const min = Number(first) || Number(p[0]); const pct = Math.max(8, Math.min(100, Math.round(min / Number(p[0]) * 100))); return ''<div style="text-align:right"><span style="font-size:26px;font-weight:800;line-height:1">'' + fmt(p[0]) + ''</span><span style="font-size:12px;opacity:.75"> kr/l</span></div><div style="margin-top:6px;height:4px;width:96px;border-radius:2px;background:rgba(255,255,255,.18);margin-left:auto"><div style="height:4px;width:'' + pct + ''%;border-radius:2px;background:#4fc3f7"></div></div>''; ]]]'
  styles:
    card:
    - padding: 10px 14px
    - border-radius: 14px
    - background: 'linear-gradient(135deg, #252b38 0%, #1b1f29 100%)'
    - border: 1px solid rgba(255,255,255,0.14)
    - color: '#fff'
    - box-shadow: none
    grid:
    - grid-template-areas: '"rank info price"'
    - grid-template-columns: 52px 1fr auto
    - align-items: center
    - column-gap: 12px
    custom_fields:
      rank:
      - justify-self: center
      info:
      - justify-self: start
      - text-align: left
      price:
      - justify-self: end
- type: custom:button-card
  entity: sensor.pepsi_max_billigst_3
  show_icon: false
  show_name: false
  show_state: false
  tap_action:
    action: url
    url_path: '[[[ const p = String(entity.state || '''').split(''|''); return p.length >= 8 ? ''https://www.tilbudsugen.dk/single/'' + p[7] : ''https://www.tilbudsugen.dk/offer/pepsi-max''; ]]]'
  custom_fields:
    rank: '[[[ const p = String(entity.state || '''').split(''|''); const chain = p[1] || ''''; if (!chain) return ''''; const key = chain.toLowerCase(); const map = {''meny'': ''meny.png'', ''rema 1000'': ''rema-1000.svg'', ''kvickly'': ''kvickly.png'', ''netto'': ''netto.png'', ''superbrugsen'': ''super-brugsen.png'', ''365discount'': ''coop365.png'', ''bilka'': ''bilka.png'', ''lidl'': ''lidl.png'', ''løvbjerg'': ''loevbjerg.png'', ''spar'': ''SPAR.jpg'', ''fakta'': ''fakta.png'', ''calle'': ''calle.png''}; const slug = key.replace(/ø/g, ''oe'').replace(/å/g, ''aa'').replace(/æ/g, ''ae'').replace(/[^a-z0-9]+/g, ''-'').replace(/^-+|-+$/g, ''''); const file = map[key] || (slug + ''.png''); const ini = chain.replace(/[^A-Za-zÆØÅæøå0-9 ]/g, '''').split('' '').filter(Boolean).slice(0, 2).map(function (w) { return w[0].toUpperCase(); }).join(''''); const edge = ''border:1px solid rgba(255,255,255,0.25);''; return ''<div style="width:46px;height:46px;border-radius:12px;background:#fff;position:relative;overflow:hidden;display:flex;align-items:center;justify-content:center;'' + edge + ''"><span style="position:absolute;color:#333;font-weight:800;font-size:15px">'' + ini + ''</span><img src="https://www.tilbudsugen.dk/chains/'' + file + ''" referrerpolicy="no-referrer" style="position:relative;width:100%;height:100%;object-fit:contain;background:#fff;padding:3px;box-sizing:border-box" onerror="this.remove()"></div>''; ]]]'
    info: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''<span style="opacity:.6">Ingen tilbud</span>''; const u = parseInt(p[3].split(''x'')[0]); const sz = String(Number(p[3].split(''x'')[1])).replace(''.'', '',''); const pack = (u > 1 ? u + '' × '' : '''') + sz + '' l''; const later = p[5] > today; return ''<div style="font-weight:700;font-size:16px;line-height:1.2">'' + p[1] + ''</div><div style="opacity:.75;font-size:12px;margin-top:2px">'' + p[2] + '' · '' + pack + '' = '' + fmt(p[4]) + '' kr</div><div style="font-size:11px;margin-top:3px;opacity:.9">'' + dd(p[5]) + ''–'' + dd(p[6]) + (later ? '' <span style="background:#ffb300;color:#111;border-radius:6px;padding:0 6px;margin-left:4px;font-weight:700">starter snart</span>'' : '''') + ''</div>''; ]]]'
    price: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''''; const first = String(((states[''sensor.pepsi_max_billigst_1''] || {}).state) || '''').split(''|'')[0]; const min = Number(first) || Number(p[0]); const pct = Math.max(8, Math.min(100, Math.round(min / Number(p[0]) * 100))); return ''<div style="text-align:right"><span style="font-size:26px;font-weight:800;line-height:1">'' + fmt(p[0]) + ''</span><span style="font-size:12px;opacity:.75"> kr/l</span></div><div style="margin-top:6px;height:4px;width:96px;border-radius:2px;background:rgba(255,255,255,.18);margin-left:auto"><div style="height:4px;width:'' + pct + ''%;border-radius:2px;background:#4fc3f7"></div></div>''; ]]]'
  styles:
    card:
    - padding: 10px 14px
    - border-radius: 14px
    - background: 'linear-gradient(135deg, #252b38 0%, #1b1f29 100%)'
    - border: 1px solid rgba(255,255,255,0.14)
    - color: '#fff'
    - box-shadow: none
    grid:
    - grid-template-areas: '"rank info price"'
    - grid-template-columns: 52px 1fr auto
    - align-items: center
    - column-gap: 12px
    custom_fields:
      rank:
      - justify-self: center
      info:
      - justify-self: start
      - text-align: left
      price:
      - justify-self: end
- type: custom:button-card
  entity: sensor.pepsi_max_billigst_4
  show_icon: false
  show_name: false
  show_state: false
  tap_action:
    action: url
    url_path: '[[[ const p = String(entity.state || '''').split(''|''); return p.length >= 8 ? ''https://www.tilbudsugen.dk/single/'' + p[7] : ''https://www.tilbudsugen.dk/offer/pepsi-max''; ]]]'
  custom_fields:
    rank: '[[[ const p = String(entity.state || '''').split(''|''); const chain = p[1] || ''''; if (!chain) return ''''; const key = chain.toLowerCase(); const map = {''meny'': ''meny.png'', ''rema 1000'': ''rema-1000.svg'', ''kvickly'': ''kvickly.png'', ''netto'': ''netto.png'', ''superbrugsen'': ''super-brugsen.png'', ''365discount'': ''coop365.png'', ''bilka'': ''bilka.png'', ''lidl'': ''lidl.png'', ''løvbjerg'': ''loevbjerg.png'', ''spar'': ''SPAR.jpg'', ''fakta'': ''fakta.png'', ''calle'': ''calle.png''}; const slug = key.replace(/ø/g, ''oe'').replace(/å/g, ''aa'').replace(/æ/g, ''ae'').replace(/[^a-z0-9]+/g, ''-'').replace(/^-+|-+$/g, ''''); const file = map[key] || (slug + ''.png''); const ini = chain.replace(/[^A-Za-zÆØÅæøå0-9 ]/g, '''').split('' '').filter(Boolean).slice(0, 2).map(function (w) { return w[0].toUpperCase(); }).join(''''); const edge = ''border:1px solid rgba(255,255,255,0.25);''; return ''<div style="width:46px;height:46px;border-radius:12px;background:#fff;position:relative;overflow:hidden;display:flex;align-items:center;justify-content:center;'' + edge + ''"><span style="position:absolute;color:#333;font-weight:800;font-size:15px">'' + ini + ''</span><img src="https://www.tilbudsugen.dk/chains/'' + file + ''" referrerpolicy="no-referrer" style="position:relative;width:100%;height:100%;object-fit:contain;background:#fff;padding:3px;box-sizing:border-box" onerror="this.remove()"></div>''; ]]]'
    info: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''<span style="opacity:.6">Ingen tilbud</span>''; const u = parseInt(p[3].split(''x'')[0]); const sz = String(Number(p[3].split(''x'')[1])).replace(''.'', '',''); const pack = (u > 1 ? u + '' × '' : '''') + sz + '' l''; const later = p[5] > today; return ''<div style="font-weight:700;font-size:16px;line-height:1.2">'' + p[1] + ''</div><div style="opacity:.75;font-size:12px;margin-top:2px">'' + p[2] + '' · '' + pack + '' = '' + fmt(p[4]) + '' kr</div><div style="font-size:11px;margin-top:3px;opacity:.9">'' + dd(p[5]) + ''–'' + dd(p[6]) + (later ? '' <span style="background:#ffb300;color:#111;border-radius:6px;padding:0 6px;margin-left:4px;font-weight:700">starter snart</span>'' : '''') + ''</div>''; ]]]'
    price: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''''; const first = String(((states[''sensor.pepsi_max_billigst_1''] || {}).state) || '''').split(''|'')[0]; const min = Number(first) || Number(p[0]); const pct = Math.max(8, Math.min(100, Math.round(min / Number(p[0]) * 100))); return ''<div style="text-align:right"><span style="font-size:26px;font-weight:800;line-height:1">'' + fmt(p[0]) + ''</span><span style="font-size:12px;opacity:.75"> kr/l</span></div><div style="margin-top:6px;height:4px;width:96px;border-radius:2px;background:rgba(255,255,255,.18);margin-left:auto"><div style="height:4px;width:'' + pct + ''%;border-radius:2px;background:#4fc3f7"></div></div>''; ]]]'
  styles:
    card:
    - padding: 10px 14px
    - border-radius: 14px
    - background: 'linear-gradient(135deg, #252b38 0%, #1b1f29 100%)'
    - border: 1px solid rgba(255,255,255,0.14)
    - color: '#fff'
    - box-shadow: none
    grid:
    - grid-template-areas: '"rank info price"'
    - grid-template-columns: 52px 1fr auto
    - align-items: center
    - column-gap: 12px
    custom_fields:
      rank:
      - justify-self: center
      info:
      - justify-self: start
      - text-align: left
      price:
      - justify-self: end
- type: custom:button-card
  entity: sensor.pepsi_max_billigst_5
  show_icon: false
  show_name: false
  show_state: false
  tap_action:
    action: url
    url_path: '[[[ const p = String(entity.state || '''').split(''|''); return p.length >= 8 ? ''https://www.tilbudsugen.dk/single/'' + p[7] : ''https://www.tilbudsugen.dk/offer/pepsi-max''; ]]]'
  custom_fields:
    rank: '[[[ const p = String(entity.state || '''').split(''|''); const chain = p[1] || ''''; if (!chain) return ''''; const key = chain.toLowerCase(); const map = {''meny'': ''meny.png'', ''rema 1000'': ''rema-1000.svg'', ''kvickly'': ''kvickly.png'', ''netto'': ''netto.png'', ''superbrugsen'': ''super-brugsen.png'', ''365discount'': ''coop365.png'', ''bilka'': ''bilka.png'', ''lidl'': ''lidl.png'', ''løvbjerg'': ''loevbjerg.png'', ''spar'': ''SPAR.jpg'', ''fakta'': ''fakta.png'', ''calle'': ''calle.png''}; const slug = key.replace(/ø/g, ''oe'').replace(/å/g, ''aa'').replace(/æ/g, ''ae'').replace(/[^a-z0-9]+/g, ''-'').replace(/^-+|-+$/g, ''''); const file = map[key] || (slug + ''.png''); const ini = chain.replace(/[^A-Za-zÆØÅæøå0-9 ]/g, '''').split('' '').filter(Boolean).slice(0, 2).map(function (w) { return w[0].toUpperCase(); }).join(''''); const edge = ''border:1px solid rgba(255,255,255,0.25);''; return ''<div style="width:46px;height:46px;border-radius:12px;background:#fff;position:relative;overflow:hidden;display:flex;align-items:center;justify-content:center;'' + edge + ''"><span style="position:absolute;color:#333;font-weight:800;font-size:15px">'' + ini + ''</span><img src="https://www.tilbudsugen.dk/chains/'' + file + ''" referrerpolicy="no-referrer" style="position:relative;width:100%;height:100%;object-fit:contain;background:#fff;padding:3px;box-sizing:border-box" onerror="this.remove()"></div>''; ]]]'
    info: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''<span style="opacity:.6">Ingen tilbud</span>''; const u = parseInt(p[3].split(''x'')[0]); const sz = String(Number(p[3].split(''x'')[1])).replace(''.'', '',''); const pack = (u > 1 ? u + '' × '' : '''') + sz + '' l''; const later = p[5] > today; return ''<div style="font-weight:700;font-size:16px;line-height:1.2">'' + p[1] + ''</div><div style="opacity:.75;font-size:12px;margin-top:2px">'' + p[2] + '' · '' + pack + '' = '' + fmt(p[4]) + '' kr</div><div style="font-size:11px;margin-top:3px;opacity:.9">'' + dd(p[5]) + ''–'' + dd(p[6]) + (later ? '' <span style="background:#ffb300;color:#111;border-radius:6px;padding:0 6px;margin-left:4px;font-weight:700">starter snart</span>'' : '''') + ''</div>''; ]]]'
    price: '[[[ const p = String(entity.state || '''').split(''|''); const ok = p.length >= 8; const fmt = (n) => (Number(n) || 0).toFixed(2).replace(''.'', '',''); const dd = (s) => s.slice(8, 10) + ''.'' + s.slice(5, 7); const t = new Date(); const today = t.getFullYear() + ''-'' + String(t.getMonth() + 1).padStart(2, ''0'') + ''-'' + String(t.getDate()).padStart(2, ''0''); if (!ok) return ''''; const first = String(((states[''sensor.pepsi_max_billigst_1''] || {}).state) || '''').split(''|'')[0]; const min = Number(first) || Number(p[0]); const pct = Math.max(8, Math.min(100, Math.round(min / Number(p[0]) * 100))); return ''<div style="text-align:right"><span style="font-size:26px;font-weight:800;line-height:1">'' + fmt(p[0]) + ''</span><span style="font-size:12px;opacity:.75"> kr/l</span></div><div style="margin-top:6px;height:4px;width:96px;border-radius:2px;background:rgba(255,255,255,.18);margin-left:auto"><div style="height:4px;width:'' + pct + ''%;border-radius:2px;background:#4fc3f7"></div></div>''; ]]]'
  styles:
    card:
    - padding: 10px 14px
    - border-radius: 14px
    - background: 'linear-gradient(135deg, #252b38 0%, #1b1f29 100%)'
    - border: 1px solid rgba(255,255,255,0.14)
    - color: '#fff'
    - box-shadow: none
    grid:
    - grid-template-areas: '"rank info price"'
    - grid-template-columns: 52px 1fr auto
    - align-items: center
    - column-gap: 12px
    custom_fields:
      rank:
      - justify-self: center
      info:
      - justify-self: start
      - text-align: left
      price:
      - justify-self: end
- type: markdown
  text_only: true
  content: '<sub>Sidst opdateret: {{ as_local(states.sensor.pepsi_max_billigst_1.last_reported).strftime(''%d.%m.%Y kl. %H:%M'') }}</sub>'
```

Kortet er lavet til visningstypen **Sektioner**, men virker også i andre.

---

## Tilpasning

- **Udeluk butikker:** ret listen `ekskluder` i skabelonen (små bogstaver, delnavn er nok). Standardlisten udelukker Løvbjerg, SPAR og ABC Lavpris. Skriv fx `'netto'` for at skjule Netto. Husk at rette på **alle fem** sensorer.
- **Andre produkter:** skift adressen i Trin 1 (fx `.../offer/coca-cola`) og ret `'pepsi'` i skabelonen til det nye mærkenavn. Ret også kortets tekster.
- **Færre eller flere rækker:** hver række er en sensor og en `custom:button-card` i kortet. Tilføj eller fjern i par.
- **Grænsehandel:** er udelukket som standard (`not it.isBorderShop`). Fjern det stykke i skabelonen for at medtage dem.
- **Mange tilbud:** siden viser kun de første 30 tilbud. De danske butikker ligger først, så de billigste er som regel med.

---

## Fejlfinding

| Problem | Løsning |
|---|---|
| Sensorerne står som `unknown` eller `unavailable` | Vent et minut efter oprettelsen. Står de stadig sådan, så kig i **Indstillinger → System → Logfiler** efter en skabelon-fejl (typisk et feltnavn, der er ændret på siden). |
| Sensorerne viser `Ingen` | Ingen tilbud opfylder kravene lige nu, eller alle butikker er udeladt i `ekskluder`. |
| Kortet viser "Custom element doesn't exist: button-card" | Button Card er ikke installeret eller ikke tilføjet som ressource. Installér den i HACS, og genindlæs siden (Ctrl+F5). |
| I stedet for et logo står der butikkens forbogstaver | Logoet findes ikke på tilbudsugen, eller din browser kan ikke hente det. Forbogstaverne er en bevidst reserve. |
| Tallene opdaterer ikke | Er "automatisk hentning" slået fra, så skal automationen fra Trin 3 være oprettet og aktiv. |

---

## Hvordan det virker

Tilbudsugens side indeholder alle tilbuddene som data i et skjult stykke JSON (`script#__NEXT_DATA__`). Scrape-sensoren læser det stykke, og skabelonen:

1. går alle tilbud igennem og beholder dem, der er Pepsi Max, solgt i liter, ikke grænsehandel og stadig gyldige,
2. sorterer efter literpris og beholder kun det billigste tilbud pr. butikskæde,
3. returnerer tilbud nummer `rang`.

Kortet læser de fem sensorers tekst og tegner rækkerne.

## Vigtigt at vide

- Denne løsning er ikke lavet af eller godkendt af Tilbudsugen, PepsiCo eller butikkerne. Navne, logoer og mærker tilhører deres ejere.
- Butikslogoerne gemmes ikke i Home Assistant. De hentes af din browser direkte fra tilbudsugen.dk, hver gang kortet vises.
- Hent ikke siden oftere end nødvendigt. To gange i døgnet er rigeligt.
