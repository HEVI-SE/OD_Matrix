# Approximerad OD-matris ur påstigningsdata — AXB/VLB (Åland)

Skattar en approximerad **ursprungs–målmatris (OD)** på hållplatsområdesnivå
(`StopArea`) för Ålands stads-/regionbussnät ur **enbart påstigningsdata** (inga
avstigningar, inga kort-ID:n). Hela analysen ligger i en körbar Jupyter-notebook:

> **[`od_matrix_estimation.ipynb`](od_matrix_estimation.ipynb)**

## Metod i korthet

Data ger *ursprung* (påstigningar) men inte *mål* (avstigningar). Utan kort-ID
kan inga individer följas. Metoden bygger därför på (jfr Kravspecifikation §2):

1. **Morgon/eftermiddag-symmetri** — em-påstigningarna per hållplats approximerar
   fm-avstigningarna (och vice versa). Det ger destinationsmarginalerna `D` som
   annars saknas. Tillämpas fönsterbaserat per linje, vilket är robust även för
   stadslinjernas slingor.
2. **Nedströms-restriktion** — man kan bara stiga av nedströms om påstigningen.
   Implementerad **radiellt** (avstånd till närmaste nav): `am` = in mot navet,
   `pm` = ut från navet. Mjukas upp automatiskt för grenade/slinglinjer där en
   hård triangel gör marginalerna oförenliga.
3. **En bytesnod** — resor genom navet är två ben (`ursprung → nav → mål`) som
   länkas via en navpivot, kalibrerad mot biljettprodukten `Bussbyte 2h`.

Marginalerna (`O` från data, `D` från symmetri) driver en **IPF/Furness**-
anpassning av en gravitationsseed inom den tillåtna nedströms-triangeln.
Resultatet bevarar de observerade påstigningsmarginalerna medan seed +
restriktionen ger strukturen.

## Indata (`data/raw/`)

| Fil | Blad | Innehåll |
|---|---|---|
| `boardings_query_result.xlsx` | `Query result` | ~21 300 rader påstigningar (linje, tur, datum, tid vid stopp, stopp, sekvens, produkt, antal). |
| `StopPointochArea.xlsx` | `Blad1` | 661 stopp → hållplatsområden + koordinater. |

Join: `boarding["Stop ID"] == stop["IsJourneyPatternPointGid"]` (verifierat
443/443), aggregering på `IsIncludedInStopAreaGid` (~302 områden).

## Köra

```bash
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace od_matrix_estimation.ipynb
# eller öppna interaktivt:  jupyter notebook od_matrix_estimation.ipynb
```

Notebooken är helt reproducerbar uppifrån och ned från de två råfilerna. Alla
parametrar (tidsfönster, dagtypsfilter, nav, byteskalibrering, seed/IPF, export)
ligger i **en parametercell överst** (§1 i notebooken).

## Utdata (`output/`, genereras vid körning)

| Fil | Innehåll |
|---|---|
| `od_matrix_stoparea.xlsx` | OD-matriser område × område, ett blad per period (`OD_am`, `OD_pm`, `OD_total`). |
| `od_matrix_{am,pm,total}.csv` | Samma matriser som CSV. |
| `od_long_format.{csv,xlsx}` | Långformat `(from_area, from_name, to_area, to_name, period, trips)` för Visum/Dynameq-import. |
| `diagnostics_report.csv` | Diagnostik (resmängd vs påstigning, byteskvot, konvergens, m.m.). |
| `validation_checks.csv` | PASS/FAIL för valideringskontrollerna (§10). |
| `fig_term_detection.png`, `fig_time_profile.png` | Terminsdetektion och bimodal tidsprofil. |

## Notebook-struktur

`0` Setup · `1` Parametrar · `2` Inläsning/join · `3` Databearbetning
(tidsparse, dagtyps-/produktfilter, riktningsdiagnostik, medelvardag) ·
`4` Marginaler (symmetri) · `5` Linje-OD (IPF) · `6` Byteskoppling ·
`7` Nät-OD · `8` Utdata · `9` Validering · `10` Kända felkällor.

## Viktiga anpassningar mot verklig data

* **Två operatörer** (AXB stads-/regionlinjer, VLB regionlinjer 2–4) = ett nät.
* **Två delnav** i centrala Mariehamn (`Mariehamn, Bussplan` + `Centrum, Nygatan`)
  hanteras som en navmängd (`HUB_AREA_GIDS`).
* **Terminsvardagar auto-detekteras** via skolkortsandel (tydlig säsong i data).
* **Slinglinjer** gör turriktning tvetydig → fönsterbaserad symmetri + radiell
  nedströms-restriktion i stället för turbaserad in-/utriktning.

## Begränsningar

Resultatet är en **approximativ** OD lämplig som startmatris för vidare
kalibrering — inte en exakt uppmätt OD. Symmetrin gäller pendel/skola men inte
ärende-/kedjeresor; enbart marginaler gör att seed styr fördelningen inom
triangeln; byteskvoten vilar på `Bussbyte 2h` (endast enkelbiljetter, uppskalad
osäkert). Se §10 i notebooken för fullständig felkälle-diskussion.
