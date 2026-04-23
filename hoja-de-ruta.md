# Hoja de ruta — nombres-nacimientos-viz

## Estado actual

App Svelte desplegada en GitHub Pages (`viz-nombres-comunes`). Dos pantallas:

1. **Entrada** — buscador de nombre + acceso al mapa de municipios (iframe)
2. **Historia** — stats personales del nombre: censo, edad media, ranking, rareza, bump chart, comparar

---

## Lógica de rareza

Escala logarítmica sobre el ranking de frecuencia del padrón 2024 (`ages.json`).

```
logPct = log(rank) / log(total_nombres) * 100
```

Con ~30.000 nombres en el padrón:

| logPct | rank aprox. | Etiqueta | Color |
|--------|-------------|----------|-------|
| 0–39   | top ~130    | muy popular | `#01f3b3` |
| 40–54  | ~130–700    | popular     | `#aaa`    |
| 55–67  | ~700–3.000  | común       | `#aaa`    |
| 68–79  | ~3.000–9.000 | poco común | `#ff922b` |
| 80–91  | ~9.000–22.000 | raro      | `#ff922b` |
| 92–100 | ~22.000–30.000 | muy raro | `#ff922b` |

Implementado en `src/lib/NameEntry.svelte` → `rarityLabel()` / `rarityColor()` / `freqRanks` Map.  
También visible en `src/lib/NameStory.svelte` en la tarjeta de stats.

---

## Datos

| Archivo | Fuente | Descripción |
|---------|--------|-------------|
| `public/data.json` | INE nacimientos | Top 100 nombres/año 2002–2023, counts y ranks |
| `public/ages.json` | INE Padrón 2024 | Frecuencia total y edad media por nombre y sexo |

Generados con `scripts/process_data.py`.

---

## Deploy

```bash
npm install
npm run deploy   # build + gh-pages -d dist
```

Repo: `viz-nombres-comunes` · Base Vite: `/viz-nombres-comunes/`  
Los fetch usan `import.meta.env.BASE_URL` para funcionar tanto en dev como en producción.

---

## Posibles mejoras futuras

- [ ] Compartir nombre por URL (query param `?nombre=HUGO&sexo=H`)
- [ ] Comparar directamente desde la URL
- [ ] Gráfico de pirámide de edad para el nombre buscado
- [ ] Versión OG image para compartir en redes
