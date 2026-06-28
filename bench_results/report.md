# Benchmark izveštaj — v1 (naivna) vs v2 (optimizovana)

- Generisano: 2026-06-28 15:23:29
- Baze: `ecommerce_v1` vs `ecommerce_v2` @ `mongodb://localhost:27017`
- Parametri: clear_cache=False, timeout=120000 ms (jedno merenje po upitu)
- Ukupno trajanje benchmarka: 826.4 s

## Rezime

| upit | v1 (ms) | v2 (ms) | ubrzanje | docs v1 | docs v2 | stage v1 | stage v2 |
|---|---|---|---|---|---|---|---|
| q6 | 32,377.3 | 2,075.4 | 15.6x | 1,000,000 | 11,455 | COLLSCAN | FETCH+IXSCAN |
| q7 | DNF | 7,457.1 | - | DNF | 1,008,000 | DNF | COLLSCAN |
| q8 | DNF | 3,025.3 | - | DNF | 1,000,000 | DNF | COLLSCAN |
| q9 | 6,190.9 | 1,723.2 | 3.6x | 1,000,000 | 135,482 | COLLSCAN | FETCH+IXSCAN |
| q10 | DNF | 5,047.6 | - | DNF | 1,000,000 | DNF | COLLSCAN |

## Komentar po upitu

### q6 — Hitna dopuna zaliha (niska zaliha vs tražnja)
*Koje proizvode hitno dopuniti — niska zaliha, a jaka realizovana tražnja?*

**v1**: COLLSCAN, 1,000,000 pregledanih dokumenata (32,377.3 ms); **v2**: FETCH+IXSCAN, 11,455 pregledanih dokumenata preko indeksa lines.product_id_1 (2,075.4 ms); ubrzanje **15.6x**

### q7 — Prekomerne (mrtve) zalihe — zarobljen kapital
*Koji artikli imaju mnogo zaliha a slabu prodaju (najviše zarobljenog kapitala)?*

**v1**: DNF/timeout — ExecutionTimeout: PlanExecutor error during aggregation :: caused by :: operation exceeded time limit; **v2**: COLLSCAN, 1,008,000 pregledanih dokumenata (7,457.1 ms)

### q8 — Kvalitet dobavljača — povrat/otkazivanje po brendu
*Koji brendovi imaju najveću stopu vraćenih/otkazanih stavki?*

**v1**: DNF/timeout — ExecutionTimeout: PlanExecutor error during aggregation :: caused by :: operation exceeded time limit; **v2**: COLLSCAN, 1,000,000 pregledanih dokumenata (3,025.3 ms)

### q9 — Rizik nestašice uz aktivnu tražnju
*Koji traženi artikli su rasprodati ili pri kraju zaliha (izgubljena prodaja)?*

**v1**: COLLSCAN, 1,000,000 pregledanih dokumenata (6,190.9 ms); **v2**: FETCH+IXSCAN, 135,482 pregledanih dokumenata preko indeksa lines.product_id_1 (1,723.2 ms); ubrzanje **3.6x**

### q10 — Sezonalni obrt — ukupna prodaja i vršni mesec
*Koliko se artikl ukupno proda i u kom mesecu je tražnja najveća (sezonsko planiranje)?*

**v1**: DNF/timeout — ExecutionTimeout: PlanExecutor error during aggregation :: caused by :: operation exceeded time limit; **v2**: COLLSCAN, 1,000,000 pregledanih dokumenata (5,047.6 ms)
