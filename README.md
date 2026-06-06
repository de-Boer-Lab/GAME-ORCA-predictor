# ORCA_1M Predictor

A [GAME](https://genomic-api-for-model-evaluation-documentation.readthedocs.io/) **Predictor** that wraps the 1 Mb [Orca](https://github.com/jzhoulab/orca) model. Given one or more DNA sequences, the Predictor returns Orca's predicted **3D genome contact map** for each sequence as a 250 × 250 matrix.

Unlike sequence-scoring Predictors that return a scalar (`point`) or a per-base array (`track`), Orca's native output is a 2D contact matrix. This Predictor therefore serves a dedicated **`interaction_matrix`** readout and can return the matrices as raw NumPy arrays over the API using `application/msgpack-numpy`.

**Underlying model:** Zhou, J. (2022). *Sequence-based modeling of three-dimensional genome architecture from kilobase to chromosome scale.* Nature Genetics 54, 725–734. [10.1038/s41588-022-01065-4](https://doi.org/10.1038/s41588-022-01065-4) · [jzhoulab/orca](https://github.com/jzhoulab/orca)

**Container:** Rui (Grey) Guo, with edits from Ishika Luthra and Satyam Priyadarshi.

---

## What it predicts

- **Model variant:** Orca **1 Mb** model, `h1esc_1m` (H1-ESC).
- **Input:** a 1,000,000 bp (1 Mb) DNA sequence per `seq_id`, one-hot encoded with `selene_sdk`. Sequences shorter/longer than the models preferred length can result in unpredictable results. 
- **Output:** a **250 × 250** contact matrix per sequence. The 1 Mb window is binned into 250 bins, giving **4 kb resolution** (1,000,000 bp ÷ 250). Values are predicted **log** fold-over-background contact scores.
- **Cell type:** H1-ESC · **Species:** *Homo sapiens* (hg38).

> The model always returns log-scale values, so `scale_prediction_actual` is reported as `"log"` regardless of the `scale` requested in the task.

---

## Supported request space

| Field | Supported value(s) | Notes |
|---|---|---|
| `readout` | `interaction_matrix` | `point` / `track` are rejected at `/predict`. |
| `type` (per task) | `conformation_chromatin` | Reported back as `type_actual: ["HI-C"]`. |
| `cell_type` (per task) | `H1-ESC` | |
| `species` (per task) | `homo_sapiens` | |
| `scale` (per task, optional) | `log` / `linear` | Echoed in the response; actual output is always `log`. |
| `prediction_ranges` | **Not supported** | Non-empty ranges are rejected (see [Limitations](#limitations)). |
| `upstream_seq` / `downstream_seq` | optional strings | Appended to each sequence before encoding. |


---

## Run the predictor

```bash
apptainer run orca_predictor.sif <HOST> <PORT>
# e.g.
apptainer run orca_predictor.sif 172.16.47.244 5000
```

In dev mode the Predictor names itself `ORCA_1M_dev`; inside the container it auto-versions from the Apptainer build date (e.g. `ORCA_1M_20251128-180629_PST`).

---

## Build the container

The Apptainer definition (`predictor.def`) builds everything: Miniconda + `mamba`, the `orca_env` environment, PyTorch (CUDA 11.8), a custom Selene fork, the Orca repo, and the Orca model weights + hg38 reference downloaded from Zenodo.

Expected host layout (the `%files` section copies `../Orca_Predictor` into the image):

```
Orca_Predictor/
├── predictor.def
├── predictor_RestAPI.py
├── orca_model.py
├── config.py
├── schema_validation.py
├── error_checking_functions.py
├── predictor_content_handler.py
└── predictor_help_message.json
```

Build:

```bash
apptainer build orca_predictor.sif predictor.def
```

---


## API

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/formats` | Lists supported request/response wire formats. |
| `GET` | `/help` | Returns model metadata from `predictor_help_message.json`. |
| `POST` | `/predict` | Runs Orca on the Evaluator sequences. |

**Wire formats** are negotiated through the `Content-Type` (request) and `Accept` (response) headers:

- **Request:** `application/json`, `application/msgpack`
- **Response:** `application/json`, `application/msgpack`, `application/msgpack-numpy`

If the Evaluator's `Accept` header includes `application/msgpack-numpy`, the 250 × 250 matrices are sent as native NumPy arrays. Otherwise they are converted to nested Python lists before JSON/MsgPack encoding.

### Example request

```json
{
  "readout": "interaction_matrix",
  "prediction_tasks": [
    {
      "name": "chr8_region",
      "type": "conformation_chromatin",
      "cell_type": "H1-ESC",
      "species": "homo_sapiens",
      "scale": "log"
    }
  ],
  "sequences": {
    "seq1": "ACGT...<1,000,000 bp>...TGCA"
  }
}
```

### Example response

```json
{
  "predictor_name": "ORCA_1M_20251128-180629_PST",
  "prediction_tasks": [
    {
      "name": "chr8_region",
      "type_requested": "conformation_chromatin",
      "type_actual": ["HI-C"],
      "cell_type_requested": "H1-ESC",
      "cell_type_actual": "H1-ESC",
      "scale_prediction_requested": "log",
      "scale_prediction_actual": "log",
      "species_requested": "homo_sapiens",
      "species_actual": "homo_sapiens",
      "predictions": {
        "seq1": [[0.12, -0.03, ...], ...]
      }
    }
  ]
}
```

`predictions[seq_id]` is a 250 × 250 matrix (list-of-lists in JSON/MsgPack, or a NumPy array under `application/msgpack-numpy`).

---

## Errors

Errors are always returned as JSON, regardless of the `Accept` header, in the form:

```json
{ "predictor_name": "ORCA_1M_20251128-180629_PST", "error": [ { "<error_key>": "<message>" } ] }
```

| HTTP | `error_key` | When |
|---|---|---|
| 400 | `bad_prediction_request` | Malformed payload — missing mandatory keys, unrecognized `readout`/`type`, bad `prediction_ranges` format. |
| 422 | `prediction_request_failed` | Request is well-formed but Orca can't fulfill it — unsupported readout/type at runtime, non-empty `prediction_ranges`, empty sequences, or invalid bases (anything outside `A/T/C/G/N`). |
| 500 | `server_error` | Unexpected backend failure. |

---

## Limitations and future work

- **Single cell type / scale.** This container serves only the 1 Mb H1-ESC model. Other Orca scales (up to whole-chromosome) and cell types (e.g. HFF) are not exposed.
- **`prediction_ranges` not yet supported.** Requests with non-empty ranges are rejected. 
- **No `linear` conversion.** Even when `scale: "linear"` is requested, the returned values remain log-scale.

---

## Citation

If you use this Predictor, please cite both the GAME framework and the underlying Orca model:

> Zhou, J. (2022). Sequence-based modeling of three-dimensional genome architecture from kilobase to chromosome scale. *Nature Genetics* 54, 725–734.

See `Orca_Predictor.technical.md` for the design rationale behind the `interaction_matrix` readout and the `msgpack-numpy` response path.
