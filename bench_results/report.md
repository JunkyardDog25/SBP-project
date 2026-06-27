# Benchmark izveštaj — v1 (naivna) vs v2 (optimizovana)

- Generisano: 2026-06-27 03:53:36
- Baze: `ecommerce_v1` vs `ecommerce_v2` @ `mongodb://localhost:27017`
- Parametri: runs=1, warmup=False, clear_cache=False, timeout=0 ms
- Ukupno trajanje benchmarka: 37.6 s

## Rezime

| upit | v1 med (ms) | v2 med (ms) | ubrzanje | docs v1 | docs v2 | stage v1 | stage v2 |
|---|---|---|---|---|---|---|---|
| q10 | n/a | 22,770.7 | - | n/a | 5,000,000 | n/a | COLLSCAN |

## Komentar po upitu

### q10 — Prihod i prosečna korpa po načinu plaćanja
*Po payment_method: prihod, broj porudžbina i prosečan broj stavki (korpa).*

**v2**: COLLSCAN, 5,000,000 pregledanih dokumenata (22,770.7 ms med)
