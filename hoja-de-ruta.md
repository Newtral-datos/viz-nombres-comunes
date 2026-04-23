# Hoja de ruta — nombres-nacimientos-viz

## Estado actual

App Svelte desplegada en GitHub Pages (`viz-nombres-comunes`). Cuatro pantallas:

1. **Entrada** — buscador de nombre + acceso al mapa de municipios (iframe) + botón explorar
2. **Historia** — stats personales del nombre: censo, edad media, ranking, rareza, bump chart, comparar
3. **Mapa** — iframe al mapa de municipios
4. **Explorar** — tabla con todos los nombres del padrón, filtrable por rareza y sexo, ordenable por cualquier columna

---

## Lógica de rareza

`logPct = log(rank) / log(total) * 100` donde `rank` y `total` son del sexo dominante del nombre (~31.000 H, ~32.000 M según Padrón 2025).

| logPct | Etiqueta    | Frecuencia aprox. | Color     |
|--------|-------------|-------------------|-----------|
| 0–39   | Muy popular | > 70.000          | `#01f3b3` |
| 40–54  | Popular     | 8.500 – 70.000    | `#aaa`    |
| 55–67  | Común       | 1.300 – 8.500     | `#aaa`    |
| 68–79  | Poco común  | 230 – 1.300       | `#ff922b` |
| 80–91  | Raro        | 50 – 230          | `#ff922b` |
| 92–100 | Muy raro    | < 50              | `#ff922b` |

Los límites de frecuencia varían ligeramente entre H y M. Implementado de forma idéntica en `NameEntry`, `NameStory` y `NameExplore`. En `NameExplore` (vista fusionada H+M) se usa siempre el ranking del sexo dominante para mantener consistencia.

---

## Datos

| Archivo | Fuente | Descripción |
|---------|--------|-------------|
| `public/data.json` | INE nacimientos | Top 100 nombres/año 2002–2024, counts y ranks |
| `public/ages.json` | INE Padrón 2025 | Frecuencia total y edad media por nombre y sexo |

Generados con `scripts/process_data.py` (paths absolutos basados en `__file__`).

---

## Deploy

```bash
npm install
npm run deploy   # build + gh-pages -d dist
```

Repo: `viz-nombres-comunes` · Base Vite: `/viz-nombres-comunes/`  
Los fetch usan `import.meta.env.BASE_URL` para funcionar tanto en dev como en producción.

---

## Mapa municipios — estado actual

- Sin cabecera de título
- Sugerencias del buscador en mayúsculas (nombres y municipios)
- Al buscar un nombre, la leyenda muestra una píldora `#01f3b3` con "Top 5 en XX municipios" (dato del campo `m` en `nombres_index.json`)
- Input de búsqueda centrado (no cambia a la izquierda al hacer focus)

---

## Posibles mejoras futuras

- [ ] Compartir nombre por URL (query param `?nombre=HUGO&sexo=H`)
- [ ] Comparar directamente desde la URL
- [ ] Gráfico de pirámide de edad para el nombre buscado
- [ ] Versión OG image para compartir en redes
