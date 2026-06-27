# E-commerce CSV → MongoDB loader

Jupyter notebook **`data_load.ipynb`** učitava e-commerce dataset (4 CSV fajla) u MongoDB u
**dve verzije** šeme, za projekat iz **optimizacije upita**:

| Verzija | Baza | Opis |
|---------|------|------|
| **Faza 1 — baseline** | `{db}_v1` | Namerno neoptimalno: 4 odvojene kolekcije (`persons`, `products`, `orders`, `orderlines`), **sve vrednosti kao stringovi**, ravna polja, podrazumevani `_id` (ObjectId), **bez indeksa**. |
| **Faza 2 — optimizovano** | `{db}_v2` | 3 kolekcije (`persons`, `products`, `orders`), orderline **ugnježden** u `orders.lines[]`, prirodni ključ kao `_id`, kastovani tipovi (int/float/datetime), poddokumenti, ciljani **indeksi**. |

Sve se učitava **streaming-om** — fajlovi se nikad ne učitavaju celi u memoriju.

## Dataset

4 CSV fajla u data direktorijumu (podrazumevano `archive/`), delimiter `;`, header red,
ne-numeričke vrednosti pod navodnicima. Prazan string ili `NULL` → `None`.

| Fajl | Redova | Napomena |
|------|--------|----------|
| `person.csv` | ~600k | |
| `product.csv` | ~8k | |
| `order.csv` | ~5M | sortiran po `order_id` (sekvencijalan) |
| `orderline.csv` | ~13M | **nije** sortiran → Faza 2 ga prvo eksterno sortira |

> `orderline.csv` nije sortiran po `order_id`, pa Faza 2 prvo radi **eksterni merge-sort**
> na disku (sortirani chunk-ovi + lazy k-way merge, ograničena memorija), a tek onda
> **streaming merge** sa `order.csv` da ugnezdi stavke u porudžbine.

## Priprema

```powershell
# 1) virtuelno okruženje + zavisnosti
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt

# 2) MongoDB preko Docker-a
docker compose up -d mongodb     # start
# docker compose down            # stop
# docker compose down -v         # stop + obriši volume
```

## Pokretanje

Otvori **`data_load.ipynb`** (VS Code / Jupyter), po potrebi izmeni `NB_*` konstante u
ćeliji **„Konfiguracija (CLI + notebook fallback)”**, pa pokreni sve ćelije
(*Run All*). Poslednja ćelija pokreće učitavanje i na kraju ispisuje rezime
(broj dokumenata po kolekciji u svakoj bazi).

### Podešavanja u ćeliji *Konfiguracija*

| Konstanta | Default | Opis |
|-----------|---------|------|
| `NB_URI` | `mongodb://localhost:27017` | MongoDB URI |
| `NB_DB` | `ecommerce` | baze postaju `{db}_v1` i `{db}_v2` |
| `NB_DATA_DIR` | `archive` | direktorijum sa CSV fajlovima |
| `NB_PHASE` | `both` | koje faze učitati: `"1"`, `"2"` ili `"both"` |
| `NB_LIMIT` | `None` | samo prvih N porudžbina (+ povezane stavke), za brz razvoj |
| `NB_BATCH_SIZE` | `10000` | veličina batch-a za `insert_many` |
| `NB_DROP` | `True` | prvo obriši ciljne kolekcije (idempotentno) |

Imena CSV fajlova su konstante na vrhu (`PERSON_CSV`, `PRODUCT_CSV`, `ORDER_CSV`,
`ORDERLINE_CSV`) — izmeni ih ako se fajlovi drugačije zovu.

> **Savet:** prvi put postavi `NB_LIMIT = 1000` pa pokreni — brzo proveri da ceo put
> (konekcija → učitavanje → indeksi → rezime) radi, pa tek onda vrati na `None` za pun load.
> Pun load je obiman (5M porudžbina / 13M stavki) i traje.

## Šema Faze 2 (optimizovano)

```text
persons:  { _id: person_id, firstname, lastname, gender, age,
            address: { street, number, unit, postalcode, city }, phone, email }

products: { _id: product_id, sku, name, description, category, subcategory, brand,
            price, cost, stock_quantity,
            dimensions: { weight_kg, length_cm, width_cm, height_cm },
            status, created_date, rating: { average, count } }

orders:   { _id: order_id, person_id, order_date, status, total_amount,
            payment_method, shipping_method, notes,
            lines: [ { line_no, product_id, quantity, unit_price, subtotal, status, notes } ] }
```

Indeksi (kreirani **posle** učitavanja, radi brzine):

* `orders`: `{order_date:1}`, `{person_id:1, order_date:1}`, `{total_amount:-1}`, `{"lines.product_id":1}`
* `products`: `{category:1}`, `{price:1}`
* `persons`: `{city:1}`

## Napomene

* `order.csv` u realnom fajlu ima `total_amount` kao **poslednju** kolonu (ne odmah posle
  `status`); kod koristi `DictReader` pa je otporan na redosled kolona.
* Prirodni ključevi (`person_id`, `product_id`, `order_id`) i reference (`orders.person_id`,
  `lines.product_id`) se u Fazi 2 kastuju u `int` da bi indeksi/sortiranja bili smisleni.
* `SORT_CHUNK_ROWS` kontroliše memoriju eksternog sortiranja: manja vrednost = manje RAM-a,
  više privremenih fajlova.
* Uz `NB_LIMIT`, Faza 1 (baseline) i dalje pun put prolazi `orderline.csv` (namerno naivna).

## Opciono: pokretanje kao skripta (CLI)

Notebook je pisan tako da radi i kao samostalna skripta sa `argparse` CLI-jem. Ako ti više
odgovara terminal nego notebook:

```powershell
jupyter nbconvert --to script data_load.ipynb
python data_load.py --data-dir archive --limit 1000 --drop
```

Argumenti: `--uri`, `--db`, `--data-dir` (obavezno), `--phase {1,2,both}`, `--limit N`,
`--batch-size`, `--drop` — isti smisao kao `NB_*` konstante iznad.
