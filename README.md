# An Open-Source, Reproducible OBIA Workflow in QGIS and Python

**High-resolution mapping of seagrass and benthic habitats in Greek coastal sites**

Nikolaos Providakis¹, Vasilis Gerakaris¹
¹ Hellenic Centre for Marine Research (HCMR), Institute of Oceanography, Anavyssos, Greece · nprov[a]hcmr[.]gr

Companion repository to the poster presented at the *I Simposio Ibérico de Cartografía y Monitorización de Praderas Marinas* (CartoGRASS 2026), Cartagena.

<!-- Zenodo badge: DOI badge will be included here once the first release is archived -->
<!-- [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.XXXXXXX.svg)](https://doi.org/10.5281/zenodo.XXXXXXX) -->

---

## What is this?

Fine-scale seagrass maps are needed for conservation assessment and monitoring under EU directives (Habitats Directive, MSFD) and for marine spatial planning. Mapping workflows differ widely between teams (pixel- vs object-based, single vs multiple classifiers, ad hoc validation), which limits comparability and transferability.

This repository accompanies a simplified, reproducible object-based image analysis (OBIA) workflow for fine-scale classification of *Posidonia oceanica* meadows and associated substrates. The workflow has been applied at 10 Greek coastal sites with free and open-source software, as a step towards a common best practice.

## Quick links

| | |
|---|---|
| 🖼️ Poster (A0, PDF) | [`poster/`](poster/) |
| 📄 Preliminary methodology report (EN) | [`docs/`](docs/) |
| 🖥️ Mar-Rast interface demo | [`demo/`](demo/): screenshots of the in-development desktop app that wraps the workflow tools (not yet released) |
| 🧰 Scripts and tools | [`scripts/`](scripts/) (to be released) |
| 📚 References | see the methodology report |

## The workflow in brief

1. **Digital and field data.** Satellite imagery (plus UAV where available) is georeferenced to the national cadastre, and layers such as bathymetry, backscatter, slope, Haralick texture, distance from shore and oblique coordinates are stacked into composite rasters.
2. **Ground truth.** ROV, drop-camera and scuba videos with GPS-tagged frames, georeferenced photogrammetric mosaics and side-scan sonar mosaics provide the expert-annotated reference points.
3. **Segmentation.** Composites are segmented into image objects (QGIS mean-shift, or a purpose-built Python segmenter based on Felzenszwalb graph segmentation; about 11 km² in 3 min instead of about 4 h).
4. **Zonal statistics.** The mean, median, standard deviation and variance of every band are calculated per object. Missing values are kept and not imputed.
5. **Learn and predict.** Random Forest (RF), Extra Trees (ET) and Histogram Gradient Boosting (HGB), all tolerant of missing values, are tuned by grid search with spatial-block cross-validation. Probabilities are calibrated, and the final class is the majority vote of the three models.
6. **Quality control.** Expert QC is applied, and all edits are logged.

Every stage is an independent, interactive tool, so that any step can be run, repeated or replaced separately, and all user-set parameters are archived.

## Demonstration area

Gramvousa peninsula, Western Crete, Greece, with a total mapped area of 40.41 km² (Balos–Kissamos 31.82 km², Falassarna 8.59 km²).

Automated classification, independent test polygons (mean of RF, ET, HGB), before manual QC:

| Site | Area | Test polygons | OA | Balanced acc. |
|---|---|---|---|---|
| Balos–Kissamos | 31.82 km² | 1,935 | 96.5 % | 96.1 % |
| Falassarna | 8.59 km² | 1,224 | 93.3 % | 83.5 % |

Kappa, F1 and per-model results are given in the methodology report.

## Repository contents

```
.
├── README.md
├── LICENSE
├── poster/      Poster (PDF) and preview image
├── docs/        Preliminary methodology report
├── demo/        Mar-Rast interface screenshots (development build)
└── scripts/     Python scripts (to be released)
```

## Roadmap

- Release of the Python scripts (segmentation, zonal statistics, split finder, optimiser, implementator)
- Open documentation per script
- PPK-corrected GCPs for centimetre-level georeferencing
- USBL-based geolocation of deep-water imagery
- Temporal change detection of meadows
- Definition of a minimum annotation effort per class and area

Peer collaboration and convergence are welcome. Please open an issue or write to the contact address above.

## How to cite

Providakis, N., Gerakaris, V. (2026). *An Open-Source, Reproducible OBIA Workflow in QGIS and Python for high-resolution mapping of seagrass and benthic habitats in Greek coastal sites.* Poster, CartoGRASS 2026, Cartagena. HCMR. https://github.com/nikprov/obia-seagrass-workflow (DOI: *to be added*)

## Licence

Documents and figures: CC BY 4.0. Code (when released): *to be confirmed (e.g. MIT or Apache-2.0)*. See [`LICENSE`](LICENSE).

## Acknowledgements and funding

This work is supported by the EU LIFE LULUCF ATLAS project. The presentation contributes to WP1/T1.2 for the dissemination of mapping methodologies, and the data generated support WP3/T3.5 for the Greek National Inventory Report (NIR).

Co-funded by the European Union.