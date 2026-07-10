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

### [Stage 02] Baseline 1: 2.5D Retrieval / Image-Text Alignment
* **Notebook chính Kaggle:** `scripts/02_retrieval_baseline_kaggle.ipynb`
* **Script đồng bộ:** `scripts/02_retrieval_baseline_kaggle.py`
* **Trạng thái:** ✅ Đã rewrite medical-grade theo Stage 00-01 cache `2757` cases.
* **Nhiệm vụ:**
  - Đọc Stage 01 cache đã PM Gate PASS: `q1_25d_uint8_224.memmap`, `q1_cache_index.csv`, `q1_clean_manifest_for_cache.csv`, `q1_cache_meta.json`.
  - Kiểm chứng 2.5D image-text alignment bằng same-split paired image-to-text retrieval.
  - Tạo train-gallery top-K retrieval cho Stage 05 RAG, nhưng không diễn giải validation/test → train là Recall exact vì split disjoint.
  - Xuất audit tables cho split/no-leakage, source_part, section completeness, SUV/FDG, clinical keywords, duplicate report hash và full cache channel stats.
* **Output bắt buộc:**
  - `vimedpet_stage02_retrieval_outputs/stage02_config.json`
  - `vimedpet_stage02_retrieval_outputs/stage02_input_audit.json`
  - `vimedpet_stage02_retrieval_outputs/retrieval_metrics.json`
  - `vimedpet_stage02_retrieval_outputs/retrieval_pm_gate.json`
  - `vimedpet_stage02_retrieval_outputs/indices/faiss_train_text.index`
  - `vimedpet_stage02_retrieval_outputs/embeddings/*_image_emb.npy`
  - `vimedpet_stage02_retrieval_outputs/embeddings/*_text_emb.npy`
  - `vimedpet_stage02_retrieval_outputs/validation_topk_retrieved.csv`
  - `vimedpet_stage02_retrieval_outputs/test_topk_retrieved.csv`
  - `vimedpet_stage02_retrieval_outputs/tables/*.csv`
* **Đánh giá:** Same-split image-to-text `Recall@1`, `Recall@5`, `Recall@10`, `MRR`, random baseline, metrics by split/source_part, failure cases, BLEU/ROUGE/BERTScore chỉ dùng như text-similarity diagnostics.
* **PM Gate:** `retrieval_pm_gate.json` phải đạt `PASS`; `stage02_input_audit.json` phải xác nhận Stage 01 PASS, `n_cases = 2757`, patient overlap zero, embeddings finite, FAISS index tồn tại.
* **Kết luận Stage 02 sau PM Review:** PASS về pipeline/output integrity nhưng là **weak retrieval baseline**; exact Recall/MRR gần random. Không được diễn giải BLEU/ROUGE/BERTScore cao là bằng chứng image-grounded PET/CT reasoning.
* **Stage 02 Fix/Audit Addendum:** Đã định lượng duplicate/near-duplicate/SUV outlier trong `vimedpet_stage02_exact_outputs/tables/stage02_engineer_loop/`: exact duplicate cross-split `3` groups / `8` cases; near-duplicate TF-IDF cosine `>=0.95` có `17` pairs, trong đó `8` cross-split; SUV `76.0` ở case `fcf882bc0dd1`, split train, source_part `PETCT_2023_2`.
* **Lưu ý y tế:** 2.5D retrieval chỉ là baseline/sanity check. Không được kết luận mô hình hiểu PET/CT 3D hoặc SUV định lượng chỉ dựa trên CT central slice, PET MIP và overlay proxy.

### [Stage 03] Baseline 2: 2.5D Generative Decoder (Mistral-7B)
* **Notebook chính Kaggle 14-16GB:** `scripts/03_mistral_7b_qlora_kaggle.ipynb`
* **Notebook fallback/debug:** `scripts/03_mistral_7b_no_bnb.ipynb`
* **Trạng thái:** ✅ Nhánh QLoRA được thiết kế để giữ Mistral-7B theo bài gốc nhưng tránh OOM fp16.
* **Nhiệm vụ:** Huấn luyện LLM sinh báo cáo dựa trên thông số hình ảnh 2.5D Stat.
* **Input (Add Data trên Kaggle):** Output đã được PM duyệt từ Stage 01.
* **Tham số quan trọng:** `MAX_SEQ_LENGTH = 512`, Mistral-7B-Instruct-v0.2, QLoRA 4-bit NF4, LoRA r=8, dropout=0.10, batch size 1, grad accumulation 8, `NUM_EPOCHS=3`, `LEARNING_RATE=5e-5`.
* **Đánh giá:** BLEU, ROUGE-L, BERTScore/multilingual embedding similarity, Report2 Keyword F1, Section completeness, SUV/FDG mention consistency, length ratio, short report rate. Bắt buộc báo duplicate-aware metrics: all-cases, exclude exact duplicate cross-split, và near-duplicate-flagged subsets.
* **Training safety:** Loss bắt buộc mask phần prompt/instruction, chỉ học phần target report sau `[/INST]`.
* **Output bắt buộc:**
  - `vimedpet_mistral_qlora_outputs/train_config.json`
  - `vimedpet_mistral_qlora_outputs/generated_reports.csv`
  - `vimedpet_mistral_qlora_outputs/eval_metrics.json`
  - `vimedpet_mistral_qlora_outputs/sample_prompts.jsonl`
  - `vimedpet_mistral_qlora_outputs/mistral_lora_adapter/adapter_config.json`
  - `vimedpet_mistral_qlora_outputs/mistral_lora_adapter/adapter_model.safetensors`
* **PM Gate:** Gửi `train_config.json`, `eval_metrics.json`, 5-10 dòng mẫu từ `generated_reports.csv`, và log `train/val/test` trước Stage 04/05.
* **Điều kiện review tối thiểu:** Notebook đọc được cache shape `(2757, 3, 224, 224)`, join đủ 2757 rows cho image/cache alignment, dùng `q1_generation_manifest.csv` `2729` rows cho supervised generation nếu yêu cầu đủ Findings/Impression, in rõ missing Findings/Impression table (`1` Findings missing, `27` Impression missing), load Mistral-7B ở 4-bit thành công, full train 3 epoch không OOM, có validation loss từng epoch, output có đủ hai section `MÔ TẢ HÌNH ẢNH` và `NHẬN ĐỊNH KẾT QUẢ`, `mean_pred_words` tăng rõ so với epoch 1, `suv_or_fdg_pred_rate` không còn gần 0, và Stage03 eval phải flag SUV outlier case `fcf882bc0dd1` / SUV `76.0` như text-level consistency case, không phải image SUV quantification.

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

---

## Decision Log — ViMed-PET/CT

### 2026-07-10 — Ghi chú audit sau Stage 00-01 Cache Streaming

*Cập nhật sau khi audit notebook `Mistral-7b/00-01-cache-streaming.ipynb` và output `Mistral-7b/00_01_cache_streaming`.*

#### Context

- Notebook đã audit: `Mistral-7b/00-01-cache-streaming.ipynb`.
- Thư mục output đã audit: `Mistral-7b/00_01_cache_streaming`.
- Kết quả tổng quát: Stage 00-01 đã hoàn tất cache cho `2757` paired PET/CT-report cases; Stage 01 PM Gate đạt `PASS`.

#### Kết quả Stage 00-01 hiện tại

- Stage 00-01 đã tạo đủ `2757` paired PET/CT-report cases.
- Stage 01 cache hoàn tất với `2757 / 2757` rows, `error_rows = 0`, `pm_gate = PASS`.
- Cache shape hiện tại là `(2757, 3, 224, 224)` với dtype `uint8`.
- Generation-ready subset hiện tại là khoảng `2729` cases do có `27` cases thiếu Impression và `1` case thiếu Findings.
- Phải phân biệt rõ hai manifest cho các mục đích khác nhau:
  - `q1_clean_manifest_for_cache.csv`: dùng cho image/cache alignment và Stage 02 retrieval, phạm vi `2757` cases.
  - `q1_generation_manifest.csv`: dùng cho supervised generation Stage 03 nếu yêu cầu đủ Findings/Impression, phạm vi khoảng `2729` cases.

#### Split audit note — patient-level split và no-leakage

- Split hiện tại theo case:
  - Train: `1930` cases, khoảng `70.0036%`.
  - Validation: `414` cases, khoảng `15.0163%`.
  - Test: `413` cases, khoảng `14.9801%`.
- Split dùng `stable_patient_group = source_part + patient_key` để tránh leakage, vì `patient_key` là local ID và có thể bị reuse giữa các `source_part`.
- Overlap theo `stable_patient_group`:
  - Train/validation: `0`.
  - Train/test: `0`.
  - Validation/test: `0`.
- Stable patient group overlap matrix bằng `0` giữa train/validation/test.
- Diagnostic audit bằng `patient_key` đơn lẻ có thể xuất hiện overlap vì patient IDs là local/reused across source_part; không được diễn giải overlap này là true patient leakage nếu chưa có global patient identifier.
- Nếu sau này tìm được global patient identifier chính thức, phải audit lại split theo global ID.

#### Các audit bắt buộc cần bổ sung trước khi đưa ra kết luận lâm sàng

##### SUV audit required

Trước khi kết luận clinical consistency hoặc viết báo cáo y tế, cần bổ sung các bảng sau:

- `SUV count by source_part`.
- `SUV count by split`.
- `SUV value distribution by source_part`.
- `SUV outlier table`, đặc biệt cần review kỹ trường hợp `SUV max = 76.0`.

Lý do: SUV là chỉ số định lượng quan trọng trong PET/CT. Nếu SUV distribution lệch giữa split/source_part hoặc regex parse nhầm số, metric Stage 03/04 có thể bị sai về mặt lâm sàng.

##### Stage 04 3D readiness notes

Hiện Stage 00-01 chưa có đủ thông tin cho kết luận 3D physical-space chuẩn y tế:

- Output hiện tại chưa có full depth distribution: `min/p25/p50/p75/max` theo split và source_part.
- Dữ liệu hiện tại là `.npy`; output Stage 00-01 chưa có DICOM/NIfTI physical metadata.
- Metadata vật lý còn thiếu gồm: spacing, origin, orientation, voxel spacing, z-spacing.
- Stage 04 3D dual-stream PET/CT phải audit depth normalization/patch sampling trước.
- Stage 04 không được tuyên bố physical-space correctness nếu chưa có metadata validation cho spacing/origin/orientation/voxel spacing.
- Nếu dataset không cung cấp metadata vật lý, phải ghi rõ limitation trong báo cáo nghiên cứu.

##### 2.5D cache limitations

2.5D cache hiện tại đúng cho Stage 02 retrieval baseline, nhưng có các giới hạn y tế cần ghi rõ:

- CT central slice chỉ là một lát giữa thân, có thể bỏ sót tổn thương ở vùng cổ, ngực, bụng, chậu hoặc các vùng rìa của axial volume.
- PET MIP giữ được projection uptake toàn thân nhưng làm mất định vị 3D.
- Overlay proxy không phải là PET/CT fusion representation đã được chuẩn hóa lâm sàng.
- Robust percentile normalization làm mất absolute SUV scale từ image representation.
- Cách biểu diễn này chấp nhận được cho Stage 02 retrieval baseline, nhưng dự án không được kết luận rằng mô hình đã hiểu sâu clinical reasoning PET/CT nếu chỉ dựa trên 2.5D cache.

##### Stage 03 caution

- Nếu Stage 03 sử dụng 2.5D cache này, mô hình có thể học report style tốt hơn là học lesion-level localization chính xác.
- Clinical metrics phải được diễn giải thận trọng.
- Stage 03 supervised generation nên dùng `q1_generation_manifest.csv`, trừ khi đã có explicit missing-section masking hoặc fallback logic cho các case thiếu Findings/Impression.

##### CT/PET/channel stats cần bổ sung

Output hiện có cache sample stats, nhưng để audit chặt hơn cần bổ sung full-cache stats:

- Full channel stats toàn cache.
- Channel stats by split.
- Channel stats by source_part.
- Blank hoặc near-blank image detection.
- Low-contrast image detection.

Các bảng đề xuất:

- `full_cache_channel_stats.csv`.
- `channel_stats_by_split.csv`.
- `channel_stats_by_source_part.csv`.
- `blank_image_review.csv`.
- `low_contrast_review.csv`.

##### Additional missing metrics vs research timeline

Các follow-up audits bắt buộc:

- Sex/age/indication/history distribution.
- Full channel stats over the entire cache, không chỉ sample stats.
- Duplicate và near-duplicate text leakage audit.
- SUV/FDG by split và source_part.
- Parser warning categories và missing-section classification.
- Section-level word count riêng cho Findings và Impression.
- Physical metadata readiness cho Stage 04 3D.

#### Các chỉ số còn thiếu so với timeline nghiên cứu

##### Mức A — Bắt buộc trước khi chốt Stage 02/03 metric

- Sex distribution by split/source_part.
- Age hoặc birth_year distribution và missingness.
- Indication completeness by split/source_part.
- Clinical history completeness by split/source_part.
- SUV/FDG by split/source_part.
- Parser warning category.
- Missing section classification.
- Findings/impression word count riêng.
- Full CT/PET/channel cache stats.

##### Mức B — Cần trước khi viết báo cáo hoặc chốt clinical consistency

- Duplicate report text hash leakage audit.
- Near-duplicate text leakage audit.
- Organ keyword matrix by split.
- Lesion keyword matrix by split.
- Negation-aware keyword audit.
- Very short/very long report review.

##### Mức C — Cần cho Stage 04/05/06 nghiên cứu sâu

- Physical metadata availability.
- Spacing/origin/orientation availability.
- Voxel spacing / z-spacing.
- SUV image calibration metadata.
- RAG evidence leakage control.
- XAI heatmap/ROI sanity design.

#### Follow-up execution order

1. Tạo audit tables Mức A để chốt data readiness chặt trước Stage 02/03.
2. Kiểm tra Stage 03 đọc đúng `q1_generation_manifest.csv` cho supervised generation.
3. Tạo duplicate/near-duplicate audit Mức B để tránh metric retrieval/generation bị inflated.
4. Ghi rõ limitation của 2.5D trong mọi báo cáo Stage 02/03.
5. Chuẩn bị Stage 04 metadata readiness trước khi thiết kế 3D dual-stream.

