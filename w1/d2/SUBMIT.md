# W1-D2: Khai ph? log, parse log v? ph?t hi?n b?t th??ng

## Submission Layout

- `notebooks/assignment.ipynb`
- `d2_pipeline.py`
- `log_analyzer.py`
- `results/top_templates.csv`
- `artifacts/outputs/`

## Nh?ng g? ?? ho?n th?nh

### Phase 1: Parse Log with Drain3

- Dataset s? d?ng: HDFS t? Loghub
- File d? li?u:
  - `data/raw/HDFS_100k.log_structured.csv`
  - `data/raw/anomaly_label.csv`
- Notebook load log c? c?u tr?c v? ??m t?ng s? d?ng
- Pipeline ?u ti?n Drain3 khi c? s?n, n?u m?i tr??ng thi?u `drain3` th? c? fallback ?? v?n ch?y ???c
- Li?t k? to?n b? template v? ??m s? d?ng m?i template
- Xu?t top-10 template ra `results/top_templates.csv`
- Ghi l?i tuning `drain_sim_th` v?i c?c gi? tr? `0.3`, `0.5`, `0.7`

### Phase 2: Anomaly Detection on Log

- T?o feature theo session/block c?a HDFS
- ?p d?ng detector ki?u Isolation Forest
- Ph?t hi?n session/template b?t th??ng
- T?nh precision / recall t? `anomaly_label.csv`

### Phase 3: Embedding + Cross-signal

- T?nh TF-IDF similarity tr?n template
- C? th? gom c?c template gi?ng nhau th?nh c?m
- Inject m?t d?ng log l? ?? ki?m tra ph?t hi?n new template

### Phase 4: Mini Log Analyzer

- `log_analyzer.py` nh?n m?t log file path
- N? in ra:
  - t?ng s? d?ng
  - s? template unique
  - top-5 templates theo count v? ratio
  - templates spike trong 1 gi? g?n nh?t
  - new templates trong 1 gi? g?n nh?t

## Output

- ?nh highlight b?t th??ng:

![HDFS anomaly highlight](artifacts/outputs/hdfs_anomaly_highlight.png)

- ?nh top template:

![HDFS top templates](artifacts/outputs/hdfs_top_templates.png)

- ?nh template count time series:

![HDFS template count time series](artifacts/outputs/hdfs_template_count_timeseries.png)

- Template exports:
  - `results/top_templates.csv`
- Log tuning: `artifacts/outputs/tuning_log.csv`
- Metrics: `artifacts/outputs/hdfs_metrics.csv`

## K?t qu? ki?m tra

Pipeline ?? ???c ki?m tra tr?n:

- `HDFS_100k.log_structured.csv`
- `HDFS_2k.log` for the mini analyzer path

T?m t?t HDFS t? structured log:

- Total lines: `104815`
- Unique templates: `19`
- Anomaly rate: `3.12%`
- Precision: `0.9667`
- Recall: `0.4633`
- F1: `0.6263`
- Top templates:
  - `Receiving block <*> src: /<*> dest: /<*>` - `23671`
  - `BLOCK* NameSystem.addStoredBlock: blockMap updated: <*> is added to <*> size <*>` - `23478`
  - `PacketResponder <*> for block <*> terminating` - `23451`
  - `Received block <*> of size <*> from /<*>` - `23447`
  - `BLOCK* NameSystem.allocateBlock:<*>` - `7940`
- Tuning `drain_sim_th`:
  - `0.3` -> `13` templates
  - `0.5` -> `13` templates
  - `0.7` -> `664` templates

## Reflection

- Drain3 r?t h?p v?i log unstructured c? m?u l?p l?i v? n? gom c?c d?ng bi?n ??ng v?o template d?ng l?i ???c.
- Ph?t hi?n template m?i quan tr?ng v? n? th??ng b?o hi?u deploy m?i, l?i m?i, ho?c h?nh vi l? ch?a t?ng th?y.
- Spike c?a template h?u ?ch khi m?t m?u l?i c? th? ??t nhi?n xu?t hi?n d?y ??c trong m?t c?a s? th?i gian.
- Metric tr? l?i c?u h?i "c?i g? ?ang sai", c?n log tr? l?i "t?i sao sai". K?t h?p c? hai s? r?t ng?n th?i gian t?m root cause.
- Structured JSON log th??ng d? query h?n, c?n plain text log th? h??ng l?i r?t nhi?u t? parsing.
- V?i HDFS l?n ch?y n?y, parser nh?m ???c c?c d?ng l?p l?i kh? t?t, nh?ng recall c?a detector v?n ? m?c trung b?nh v? detector ??n gi?n ch?a b?t h?t m?i failure mode.

## Nh?n X?t Bonus

V?i log ki?u Docker-style c?a c?ng m?t service, log JSON c? l?i th? r?t r? v? c?c field nh? `timestamp`, `level`, `message`, `service`, `user`, `order` ?? c? c?u tr?c s?n n?n d? ??m, d? l?c v? ?t template h?n. Ng??c l?i, plain text c?n Drain3 ho?c regex parser ?? gom c?c d?ng bi?n ??ng v?o c?ng m?t template. Trong th? nghi?m c?a em, regex parser cho ra output ?n v? d? ki?m so?t theo format ?? bi?t, c?n Drain3 linh ho?t h?n khi format log thay ??i ho?c khi xu?t hi?n pattern m?i ngo?i d? ?o?n. V? v?y, n?u h? th?ng ?? log ???c JSON th? n?n ?u ti?n structured log; c?n n?u l? log c? ho?c l?n nhi?u format th? Drain3 v?n l? l?a ch?n an to?n h?n.
