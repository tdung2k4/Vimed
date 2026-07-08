# ViMed-PET/CT Model Pipeline Roadmap
*Tài liệu dành cho Kaggle Execution - Cập nhật theo Report 2*

Tài liệu này đóng vai trò kim chỉ nam, liệt kê toàn bộ tiến trình từ dữ liệu thô (Raw) đến XAI. Mỗi bước (Stage) phải sử dụng **kết quả đánh giá của Stage trước đó** làm cơ sở, tuyệt đối không nhảy cóc.

## Quy tắc output bắt buộc

- Mỗi Stage phải in danh sách output cuối notebook kèm dung lượng file.
- Mỗi Stage phải có metadata JSON để PM review trước khi chạy Stage kế tiếp.
- Không dùng output tạm hoặc file thiếu PM gate làm input cho Stage sau.
- Nếu có lỗi, phải gửi cả file lỗi CSV/JSON; không chỉ gửi ảnh chụp màn hình.
- Với dữ liệu y tế, mọi case bị loại phải có audit trail, không được xóa âm thầm.

## Cấu Trúc Notebooks & Hướng Dẫn Chạy

Pipeline bao gồm 6 giai đoạn chính (bám sát Cell 30 của bài báo gốc).

### [Stage 00] Archive Listing & Manifest Creation
* **Notebook:** `scripts/00_archive_listing_manifest_clean.ipynb`
* **Trạng thái:** ✅ Đã refactor medical-grade, không dùng nguyên code `00` của hannhu.
* **Nhiệm vụ:**
  - Không full-extract PET/CT image; chỉ quét archive bằng `7z l -slt`.
  - Chỉ extract JSON report để parse `Findings`/`Impression`.
  - Robust parser cho nhiều biến thể key: `Mô tả hình ảnh`, `Findings`, `Nhận định kết quả`, `Impression`, `Kết luận`.
  - Ghép case bằng `source_part + patient + month/day + stable case hash` để tránh mất multi-exam patient.
  - Tạo patient-level split 70/15/15 và kiểm tra leakage.
* **Output bắt buộc:**
  - `q1_all_cases_manifest_with_flags.csv`
  - `q1_clean_manifest.csv`
  - `q1_clean_manifest_for_cache.csv`
  - `q1_manifest_summary.json`
  - `tables/01_section_completeness.csv`
  - `tables/02_keyword_counts.csv`
  - `tables/03_suv_stats.csv`
  - `tables/04_max_token_readiness.csv`
* **PM Gate:** Sau khi chạy Stage 00 trên Kaggle, phải gửi `q1_manifest_summary.json` để review trước khi chạy Stage 01.
* **Lưu ý:** Report 2 chỉ định `MAX_SEQ_LENGTH = 512`; không đổi sang 716 nếu chưa có bằng chứng mới.

### [Stage 01] 2.5D Cache Streaming
* **Notebook:** `scripts/01_build_25d_cache_streaming.ipynb`
* **Trạng thái:** ✅ Đã tạo notebook chính thức sau khi Stage 00 pass PM Gate.
* **Nhiệm vụ:**
  - Đọc `q1_clean_manifest_for_cache.csv` từ output Stage 00 đã duyệt.
  - Không full-extract toàn bộ dataset.
  - Stream extract chỉ CT/PET `.npy` cần dùng theo từng `source_part`.
  - Tạo cache `np.memmap` uint8 shape `(N, 3, 224, 224)`:
    - channel 0: CT central slice
    - channel 1: PET MIP
    - channel 2: CT/PET overlay proxy
* **Input Stage 00 trên Kaggle:** Add Data notebook output cũ, ví dụ `/kaggle/input/notebooks/tundng111/00-archive-listing-manifest-clean`.
* **Input raw archives:** Add Data 3 dataset raw Part 1/2/3.
* **Output bắt buộc:**
  - `01_cache/q1_25d_uint8_224.memmap`
  - `01_cache/q1_cache_index.csv`
  - `01_cache/q1_clean_manifest_for_cache.csv`
  - `01_cache/q1_cache_meta.json`
  - `01_cache/q1_cache_errors.csv`
  - `01_cache/tables/01_cache_sample_stats.csv`
  - `01_cache/tables/02_cache_split_status.csv`
* **Schema bắt buộc của Stage 01:**
  - `q1_cache_meta.json` phải có `pm_gate`, `shape`, `dtype`, `ok_rows`, `error_rows`, `cache_path`.
  - `shape` phải là `[N, 3, 224, 224]` với `N = clean_cases`.
  - `q1_cache_index.csv` phải có `case_id`, `row_index`, `status`, `ct_path`, `pet_path`, `report_path`.
  - `q1_cache_errors.csv` phải có header; nếu không lỗi thì số dòng data bằng 0.
* **PM Gate:** Sau khi chạy Stage 01 trên Kaggle, gửi `q1_cache_meta.json` và `q1_cache_errors.csv` trước khi chạy Stage 03.
* **Điều kiện PASS:** `pm_gate = PASS`, `error_rows = 0`, `ok_rows = N`, memmap bytes đúng `N * 3 * 224 * 224`.

### [Stage 02] Baseline 1: Retrieval
* **Notebook:** *(Chờ phát triển)*
* **Nhiệm vụ:** Đánh giá Image-Text Alignment trước khi huấn luyện LLM thông qua tính toán tương đồng ngữ nghĩa.
* **Đánh giá:** Nếu Recall@1 và MRR > Baseline, tiếp tục sang Generative.

### [Stage 03] Baseline 2: 2.5D Generative Decoder (Mistral-7B)
* **Notebook chính Kaggle 14-16GB:** `scripts/03_mistral_7b_qlora_kaggle.ipynb`
* **Notebook fallback/debug:** `scripts/03_mistral_7b_no_bnb.ipynb`
* **Trạng thái:** ✅ Nhánh QLoRA được thiết kế để giữ Mistral-7B theo bài gốc nhưng tránh OOM fp16.
* **Nhiệm vụ:** Huấn luyện LLM sinh báo cáo dựa trên thông số hình ảnh 2.5D Stat.
* **Input (Add Data trên Kaggle):** Output đã được PM duyệt từ Stage 01.
* **Tham số quan trọng:** `MAX_SEQ_LENGTH = 512`, Mistral-7B-Instruct-v0.2, QLoRA 4-bit NF4, LoRA r=8, batch size 1.
* **Đánh giá:** BLEU, ROUGE-L, Report2 Keyword F1, Section completeness, SUV mention rate.
* **Training safety:** Loss bắt buộc mask phần prompt/instruction, chỉ học phần target report sau `[/INST]`.
* **Output bắt buộc:**
  - `vimedpet_mistral_qlora_outputs/train_config.json`
  - `vimedpet_mistral_qlora_outputs/generated_reports.csv`
  - `vimedpet_mistral_qlora_outputs/eval_metrics.json`
  - `vimedpet_mistral_qlora_outputs/sample_prompts.jsonl`
  - `vimedpet_mistral_qlora_outputs/mistral_lora_adapter/adapter_config.json`
  - `vimedpet_mistral_qlora_outputs/mistral_lora_adapter/adapter_model.safetensors`
* **PM Gate:** Gửi `train_config.json`, `eval_metrics.json`, 5-10 dòng mẫu từ `generated_reports.csv`, và log `train/val/test` trước Stage 04/05.
* **Điều kiện review tối thiểu:** Notebook đọc được cache shape `(2729, 3, 224, 224)`, join đủ 2729 rows, load Mistral-7B ở 4-bit thành công, train smoke ít nhất 10 steps không OOM, output có đủ hai section `MÔ TẢ HÌNH ẢNH` và `NHẬN ĐỊNH KẾT QUẢ` ở mẫu sinh.

### [Stage 04] Main Target: 3D Dual-Stream PET/CT Encoder
* **Notebook:** *(Chờ phát triển)*
* **Nhiệm vụ:** Xử lý nguyên khối 3D thay vì MIP 2.5D. Cần xử lý Depth Normalization / Patch Sampling.
* **Kỳ vọng:** **SUV Consistency** và Clinical Precision tăng mạnh so với Stage 03.

### [Stage 05] RAG: Retrieval-Augmented Generation
* **Notebook:** *(Chờ phát triển)*
* **Nhiệm vụ:** Dùng retrieval engine (từ Stage 02) lấy Top-K báo cáo tương đồng nhất nạp vào ngữ cảnh của Mistral-7B (Stage 03/04).
* **Mục tiêu:** Giảm Hallucination (sinh ra số SUV ảo).

### [Stage 06] XAI: Interpretability
* **Notebook:** *(Chờ phát triển)*
* **Nhiệm vụ:** Chiết xuất bản đồ sự chú ý (Attention Heatmaps / Grad-CAM). Xác minh xem khi mô hình sinh ra từ "tổn thương ở phổi", Heatmap có thực sự focus vào phổi trên hình hay không.
