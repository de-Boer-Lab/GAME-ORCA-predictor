# ORCA_1M Predictor — Technical Notes

This document covers the design decisions specific to the Orca Predictor: how Orca's contact-map output is mapped onto the GAME API, the new `interaction_matrix` readout, the `msgpack-numpy` response path, and the current handling of flanks and prediction ranges. For build/run/usage instructions see the top-level [`README.md`](README.md).

---

## 1. Orca Readout


The 1 Mb model (`h1esc_1m`) takes a 1 Mb sequence and predicts a **2D contact map**: a 250 × 250 matrix where entry `(i, j)` is the predicted interaction frequency between bin `i` and bin `j`. Each value in the matrix represents a 4Kbp by 4Kbp region in log scale (log fold over distance).

Input length: The 1 Mb model expects sequences of exactly 1,000,000 bp. The Predictor does not pad or trim to 1 Mb itself, so the Evaluator is responsible for supplying sequences of the expected length (flanks are applied before encoding and count toward that length). Sequences shorter or longer than 1 Mb can produce unreliable results and are not recommended at this stage (see future work below).
---

## 3. The `msgpack-numpy` response path

To create efficient message passing for large matricies between Evaluators and Predictions, the ORCA Predictor includes functionality for `msgpack-numpy` responses. The predictions are coded as numpy arrays if Evaluators can handle that response format, otherwise the Predictions are converted to lists. 

The following patch in the code allows for this functionality:

```python
import msgpack_numpy as m
m.patch()
```

Format selection using the request headers. 

1. `application/msgpack-numpy` → arrays sent as native NumPy (most efficient).
2. `application/msgpack` → standard MessagePack.
3. `application/json` → default fallback.

Serializing of predictions is done based on the Evaluator accept header. 

```python
if "application/msgpack-numpy" in accept_header:
    orca_preds_serializable = orca_preds
else:
    orca_preds_serializable = {k: v.tolist() for k, v in orca_preds.items()}
```

## 7. File map

| File | Responsibility |
|---|---|
| `predictor_RestAPI.py` | Flask app; `/formats`, `/help`, `/predict`; orchestrates validate → preprocess → predict → encode; assembles the per-task response. |
| `orca_model.py` | Loads the Orca 1 Mb resources and `h1esc_1m`; encodes sequences and runs the model; returns `{seq_id: 250×250 ndarray}`. |
| `schema_validation.py` | `validate_request_payload` (schema) and `preprocess_data` (flanks, alphabet, model-specific checks). |
| `error_checking_functions.py` | `APIError` hierarchy and all individual field checks. |
| `predictor_content_handler.py` | `decode_request` / `encode_response`, including the `msgpack-numpy` path. |
| `config.py` | Predictor name + versioning, supported wire formats, help-file path. |
| `predictor_help_message.json` | Metadata served by `/help` (model, version, cell types, species, authors, input size). |
| `predictor.def` | Apptainer recipe: `orca_env`, custom Selene fork, Orca repo + Zenodo weights, run/start scripts. |

---

## 8. Future work

- Implement `prediction_ranges`
- Allow Predictor to use HFF cell type included in it's training data
- Encapsulate other resolution ORCA models (e.g up to 256Mbp) to allow for flexibility in sequence size
- Encapsulate models HFF cell type
- Implement functionality for custom sequence size prediction


