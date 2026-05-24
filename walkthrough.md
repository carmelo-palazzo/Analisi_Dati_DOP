# Walkthrough — Analisi raggi di diffrazione elettronica

## Obiettivo
Estrarre i raggi degli anelli di diffrazione da foto dello schermo fluorescente, con incertezze, per 5 tensioni di accelerazione (3.4–4.5 kV).

## Struttura del notebook [analisi_raggio.ipynb](file:///Users/carmelo/Desktop/Universit%C3%A0/III%20ANNO/Lab%20di%20Fisica%20Moderna/Esperienze/Dualismo_Onda-Particella/Analisi_Dati/analisi_raggio.ipynb)

### Cella 1 — Funzioni e librerie

Contiene tutte le funzioni riutilizzabili. Nessun output diretto.

---

#### `find_green_circle(image_gray)`
**Scopo**: Rilevare il bordo dello schermo fluorescente verde (Ø = 100 mm) per stabilire la scala mm/px.

**Algoritmo**:
1. Soglia binaria a intensità > 20 → separa disco verde dallo sfondo nero
2. `ndimage.binary_fill_holes` → riempie buchi interni
3. `ndimage.label` → etichetta regioni connesse, seleziona la più grande
4. Erosione (3 iterazioni) → differenza con maschera originale dà i **pixel di bordo**
5. **Fit cerchio ai minimi quadrati** sui ~11.000 pixel di bordo:
   - Modello: `(x−a)² + (y−b)² = R²`
   - Riformulato come sistema lineare: `2ax + 2by − c = x² + y²`
   - Risolto con `np.linalg.lstsq`

> [!IMPORTANT]
> Il centro del disco verde **NON coincide** con il centro dello spot di diffrazione (~7 mm di offset). La calibrazione usa il disco verde, il profilo radiale usa il centro dello spot.

---

#### `find_center(image)`
**Scopo**: Trovare il centro sub-pixel dello spot di diffrazione.

**Algoritmo**:
1. `np.argmax` → stima iniziale grossolana (pixel più luminoso)
2. Estrae una ROI 60×60 px centrata sull'argmax
3. Fit gaussiano 2D (`curve_fit`) sulla ROI:
   - `I(x,y) = A·exp(−((x−x₀)²/2σ_x² + (y−y₀)²/2σ_y²)) + offset`
4. Centro sub-pixel dalla matrice di covarianza (incertezza ~0.05 px)

> [!NOTE]
> L'argmax è inaffidabile su spot saturati (clipping, hot pixels). Il fit gaussiano è robusto perché usa tutta la distribuzione spaziale.

---

#### `radial_profile(image, center)`
**Scopo**: Calcolare il profilo medio azimutale dell'intensità in funzione del raggio.

**Algoritmo**:
1. Per ogni pixel, calcola la distanza euclidea dal centro
2. Arrotonda a intero → bin di raggio
3. `np.bincount` con pesi = intensità → somma per bin
4. `np.bincount` senza pesi → conteggio per bin
5. Media = somma / conteggio

> [!TIP]
> Rispetto al profilo su singola riga, il profilo azimutale media su **centinaia di pixel** per ogni raggio, aumentando drasticamente il rapporto segnale/rumore.

---

#### `find_half_max_radii(r_data, I_data)`
**Scopo**: Trovare il raggio interno e esterno di ciascun anello al livello di semi-massimo.

**Algoritmo**:
1. Trova il picco (`argmax`)
2. Calcola la **baseline indipendente per lato**:
   - Lato interno: minimo dell'intensità tra inizio finestra e picco
   - Lato esterno: minimo tra picco e fine finestra
3. Semi-massimo per lato: `hm = baseline + (peak − baseline) / 2`
4. Interpolazione lineare per trovare il crossing esatto

> [!IMPORTANT]
> Le baseline separate sono essenziali per i ring che stanno sulla coda dello spot centrale (es. Ring 1 a 4.1–4.5 kV), dove il lato interno ha un fondo molto più alto del lato esterno.

---

#### `analyze_image(file_path, voltage_kV)`
Funzione wrapper che esegue l'intera pipeline su un'immagine:
1. Calibrazione verde → `mm_per_px`
2. Centro spot → `(cx, cy)`
3. Profilo radiale → smoothing Savitzky-Golay (finestra 31, poly 3)
4. `find_peaks` (prominenza > 0.5, larghezza > 5, distanza > 20 px)
5. Per ogni picco: fit gaussiano (con bounds su σ ∈ [0.5, 30] px) + half-max radii
6. Output: `r_medio = (r_int + r_ext) / 2` e `Δr = (r_ext − r_int) / 2`

---

### Cella 2 — Loop su tutte le immagini

Itera su 5 file ([3.4_kV.jpeg](file:///Users/carmelo/Desktop/Universit%C3%A0/III%20ANNO/Lab%20di%20Fisica%20Moderna/Esperienze/Dualismo_Onda-Particella/Foto/3.4_kV.jpeg) → `4.5_kV.jpeg`), chiama `analyze_image()` per ciascuno, e genera i grafici (profilo radiale + overlay cerchi sull'immagine).

### Cella 3 — Tabella riassuntiva

Stampa:
- Tabella dettagliata: V, ring, r_int, r_ext, r_medio, Δr (in mm)
- Tabella compatta: V, r₁±Δr₁, r₂±Δr₂
- Rapporto r₂/r₁ per ciascuna tensione (atteso √3 ≈ 1.73)

## Bug corretti rispetto al codice originale ([codice.ipynb](file:///Users/carmelo/Desktop/Universit%C3%A0/III%20ANNO/Lab%20di%20Fisica%20Moderna/Esperienze/Dualismo_Onda-Particella/Analisi_Dati/codice.ipynb))

| Bug | Originale | Corretto |
|-----|-----------|----------|
| Indice riga/colonna | `matrice_bn[coords[1],:]` (colonna come riga) | `matrice_bn[coords[0],:]` + profilo radiale |
| `window_length` | `0.01` (float → crash) | `31` (intero dispari) |
| Dimensioni array | `linspace(0, max)` = 50 punti vs 1259 nell'immagine | Profilo radiale con dimensione coerente |
| Centro | `argmax` (fragile su saturazione) | Fit gaussiano 2D sub-pixel |
| Scala fisica | Non definita | Disco verde Ø = 100 mm |
