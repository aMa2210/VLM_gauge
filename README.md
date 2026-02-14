## VLM Gauge – Vision-Language Models for Analog Gauge Reading

This repository explores and evaluates vision-language models on **industrial analog gauge reading** tasks.  
Given gauge videos, the model outputs time-series readings, which are then aligned with encoder-verified ground truth to compute MAE / RMSE / TMS and quantify performance in industrial metrology scenarios.

---

### Features

- **Two gauge types**
  - `Analog Dial Gauge`
  - `Analog Depth Gauge`
- **Auto reading from gauge videos**  
  Uses Gemini 3 models to read values from the pointer, scale marks, and on-screen digital chronometer, and outputs a timestamped reading sequence.
- **Time-aligned accuracy evaluation**
  - **MAE** (Mean Absolute Error)
  - **RMSE** (Root Mean Squared Error)
  - **TMS** (Trend Matching Score)
- **Unified analysis and summary scripts**
  - Parse JSON from raw model responses
  - Align with ground truth
  - Export detailed and summary metrics to Excel

---

### Project Structure


- `Dataset/`
  - `1.- Analog Dial Gauge/`: dial gauge videos (`.mp4`)
  - `2.- Analog Depth Gauge/`: depth gauge videos (`.mp4`)
  - `1_label/`: ground truth Excel files for dial gauges (with `Metadata` / `Ground_Truth` sheets)
  - `2_label/`: ground truth Excel files for depth gauges
- `Results/`
  - `1.- Analog Dial Gauge<model_name>_naive/`: raw model outputs for dial gauges (`*_Raw_Results.xlsx`)
  - `2.- Analog Depth Gauge<model_name>_naive/`: raw model outputs for depth gauges
- `Analysed/`
  - `Analysis_Summary_*.xlsx`: per-task analysis summary
  - `Combined_Analysis_Summary_*.xlsx`: cross-task combined analysis
- Notebooks / scripts (in repo root):
  - `Call_Gemini.ipynb`: run Gemini on **analog dial gauge** videos
  - `Call_Gemini_2.ipynb`: run Gemini on **analog depth gauge** videos
  - `Analyse.ipynb`: compute evaluation metrics from `Results/` and `Dataset/*_label`
  - `Extract_video_metaInfo.ipynb`: get FPS, resolution, duration, etc. using OpenCV

---

### Data Preparation

1. **Organize videos**

   - Place dial gauge videos in:  
     `Dataset/1.- Analog Dial Gauge/`
   - Place depth gauge videos in:  
     `Dataset/2.- Analog Depth Gauge/`

2. **Prepare ground truth labels**

   For each video sequence (e.g., `V1_5 divisions_per_second`), create:

   - `Dataset/1_label/V1_5 divisions_per_second.xlsx` (dial)
   - `Dataset/2_label/V1_5 divisions_per_second.xlsx` (depth)

   Each Excel file should contain:

   - `Metadata` sheet: gauge type, unit, range, graduation interval, etc.
     - Columns like:
       - `type`
       - `unit`
       - `manufacturer_range_min`
       - `manufacturer_range_max`
       - `graduation_interval`
   - `Ground_Truth` sheet: time-series labels
     - `timestamp_ms`: time (ms)
     - `encoder_verified_reading`: encoder-verified reading

3. **Set Gemini API key**

   Set the environment variable:

   ```bash
   # Windows PowerShell example
   setx GOOGLE_API_KEY "YOUR_API_KEY_HERE"
   ```

   The notebooks use:

   ```python
   import os
   from google import genai

   client = genai.Client(api_key=os.environ["GOOGLE_API_KEY"])
   ```

---

### Usage

#### 1. Run Gemini on analog dial gauge videos

Open `Call_Gemini.ipynb` and run all cells:

- Load metadata from `Dataset/1_label/...` (`Metadata` sheet) to get:
  - `gauge_type`, `unit`
  - `manufacturer_range_min`, `manufacturer_range_max`
  - `graduation_interval`
- Enumerate `Dataset/1.- Analog Dial Gauge/` for `.mp4` videos.
- For each video:
  - Upload the video via `client.files.upload`.
  - Build a prompt including:
    - Gauge metadata (type, unit, range, graduation interval)
    - Sampling rule:
      - Use the on-screen digital chronometer as absolute time reference.
      - Start from the first integer multiple of 200 ms.
      - Output one reading every 200 ms (e.g., 5800, 6000, 6200, ...).
    - Output format: **JSON array only**, each object:

      ```text
      {"ts_ms": integer, "reading": float, "confidence": float}
      ```

  - Call:

    ```python
    response = client.models.generate_content(
        model=model_name,
        contents=[myfile, message],
        config=types.GenerateContentConfig(temperature=0.0),
    )
    ```

  - Save results to:

    ```text
    Results/1.- Analog Dial Gauge<model_name>_naive/<seq_id>_Raw_Results.xlsx
    ```

    with columns like:
    - `sequence_id`
    - `raw_model_response`
    - `total_duration_sec`
    - `model_name`

#### 2. Run Gemini on analog depth gauge videos

Open `Call_Gemini_2.ipynb` and repeat the same process for depth gauges:

- Use:
  - `Dataset/2.- Analog Depth Gauge/`
  - `Dataset/2_label/`
- Save to:
  - `Results/2.- Analog Depth Gauge<model_name>_naive/`

The prompt structure is analogous (role: Industrial Metrology Assistant, 200 ms sampling, JSON output, etc.).

#### 3. Evaluate and analyze

Open `Analyse.ipynb`. The main steps are:

1. **Load raw model outputs**

   - Iterate over `RESULTS_DIR` (e.g., `Results/2.- Analog Depth Gaugegemini-3-pro-preview_naive`).
   - For each `*_Raw_Results.xlsx`:
     - Read `raw_model_response` from the first row.
     - Use `extract_json_from_text` to:
       - Strip optional ```json fences.
       - Parse JSON into a Python list and then into a DataFrame `vlm_df` with columns like `ts_ms`, `reading`, `confidence`.

2. **Load ground truth**

   - Construct `gt_path = os.path.join(GT_DIR, f"{seq_id}.xlsx")`.
   - Read `Ground_Truth` sheet:

     ```python
     gt_df = pd.read_excel(gt_path, sheet_name="Ground_Truth")
     ```

3. **Align and compute metrics**

   - Normalize column names (lowercase, trimmed).
   - Sort:
     - `gt_df` by `timestamp_ms`
     - `vlm_df` by `ts_ms`
   - Time-align using nearest-neighbor merge with tolerance 200 ms:

     ```python
     merged = pd.merge_asof(
         vlm_df,
         gt_df,
         left_on="ts_ms",
         right_on="timestamp_ms",
         direction="nearest",
         tolerance=200,
     ).dropna(subset=["encoder_verified_reading", "reading"])
     ```

   - Extract arrays:
     - `rvlm` = model readings
     - `rgt`  = ground truth readings
   - Compute:
     - **MAE** = mean absolute error
     - **RMSE** = root mean squared error
     - **TMS**:
       - Compute Δr_vlm, Δr_gt
       - Compare `sign(diff_vlm)` vs `sign(diff_gt)`
       - Take mean of matches

4. **Aggregate and export**

   - For each sequence, record:
     - `sequence_id`
     - `Valid_Samples`
     - `MAE`
     - `RMSE`
     - `TMS`
   - Compute global statistics:
     - Mean and standard deviation of `MAE`, `RMSE`, `TMS`
   - Save to Excel, e.g.:

     - `Analysed/Analysis_Summary_Depth_pro_naive.xlsx`
       - `Summary_Statistics`: `Metric`, `Mean ± SD`
       - `Detailed_Metrics`: per-sequence metrics

   - There is also a configuration list `TASK_CONFIGS` for running multi-task combined analysis and writing `Combined_Analysis_Summary_*.xlsx`.

#### 4. (Optional) Inspect video metadata

`Extract_video_metaInfo.ipynb` uses OpenCV to read:

- FPS
- Resolution
- Duration in ms

Example output:

- `fps: 30.0`
- `resolution: 1080x1920`
- `duration_ms: 65489`

This is useful for sanity-checking sampling intervals and dataset quality.

---

### Metrics

- **MAE (Mean Absolute Error)**  
  Average absolute difference between model predictions and ground truth; lower is better.
- **RMSE (Root Mean Squared Error)**  
  Square-root of mean squared error; more sensitive to large errors.
- **TMS (Trend Matching Score)**  
  Fraction of time steps where the sign of the change matches between model and ground truth, reflecting how well the model tracks the true trend:

  \[
  \text{TMS} = \frac{1}{N-1} \sum_{i} \mathbb{1}\left[ \text{sign}(r_{vlm,i+1}-r_{vlm,i}) = \text{sign}(r_{gt,i+1}-r_{gt,i}) \right]
  \]

---

### Typical Workflow

1. Prepare videos and label Excel files under `Dataset/*` as described.
2. Set `GOOGLE_API_KEY` in the environment.
3. Run:
   - `Call_Gemini.ipynb` for analog dial gauges.
   - `Call_Gemini_2.ipynb` for analog depth gauges.
4. Run `Analyse.ipynb` to compute and export MAE / RMSE / TMS to `Analysed/*.xlsx`.
5. Inspect Excel reports for per-sequence and overall performance.


