# Giải thích toàn diện metric, PM Gate và lựa chọn V1–V3

## 1. Kết luận cần đọc trước

> [!CAUTION]
> **Không version nào đạt điều kiện dùng lâm sàng.** Cả V1, V2 và V3 đều `PM Gate = FAIL`, chủ yếu do `clinical_text_safety_gate = FAIL`. Clinical readiness của cả ba là `NOT_ESTABLISHED`.

### Khuyến nghị lựa chọn

- **Nếu buộc phải chọn một version để tiếp tục sửa và nghiên cứu:** chọn **V1**.
- **Không chọn V1 để sử dụng cho bác sĩ/bệnh nhân ở trạng thái hiện tại.**
- Không chọn V2 chỉ vì BLEU/ROUGE/BERTScore cao hơn: V2 có abnormal-to-normal flags, SUV hallucination và lỗi tiếng Việt cao nhất.
- V3 là lựa chọn phụ nếu ưu tiên abnormal-to-normal thấp nhất, nhưng SUV omission, truncation và raw text disorder vẫn nghiêm trọng.

### Vì sao tạm chọn V1 để sửa tiếp?

1. Keyword precision/recall/F1 cao nhất.
2. Organ F1 cao nhất.
3. Finding-status F1 cao nhất.
4. Clean-Vietnamese screening tốt nhất: 412/414 cases.
5. Không có SUV hallucination flag.
6. V1 cao hơn V3 có ý nghĩa ở keyword recall/F1 và organ F1.
7. Nhược điểm V1 vẫn rất nặng: 360 SUV-missed, 385 likely-truncated, 372 critical-error-any và raw text loạn nghĩa. Vì vậy lựa chọn V1 chỉ là **development candidate**, không phải model đã đạt.

---

## 2. Phạm vi quét

Đã quét toàn bộ [Mistral-7b](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b):

- **356 files**;
- **22 Markdown files**;
- notebook cache Stage00–01;
- ba adapter packages V1/V2/V3;
- training config/log/checkpoints;
- ba evaluation notebooks;
- 63 evaluation JSON files, 21 file/version;
- generated reports và case-level tables;
- báo cáo audit trước đó.

Inventory đầy đủ: [mistral7b_complete_file_inventory.csv](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/mistral7b_complete_file_inventory.csv).

---

## 3. Làm rõ 2.729, 2.757 và 1.930 cases

> [!IMPORTANT]
> **Không có bằng chứng version nào “train trên 2.729 cases”.** Cách gọi đó trộn ba khái niệm khác nhau.

| Khái niệm | Số lượng | Ý nghĩa đúng |
|---|---:|---|
| Total paired/cache scope | 2.757 | Toàn bộ dataset gồm train, validation, test |
| Full-report/generation-ready | 2.729 | Có đủ Findings và Impression |
| Partial-report | 28 | 27 thiếu Impression, 1 thiếu Findings |
| Train split | **1.930** | Rows thuộc training split và được config ghi là `train_rows` |
| Validation | 414 | Không phải training rows |
| Test | 413 | Không phải training rows |

`1.930 + 414 + 413 = 2.757`.

### Mapping chính xác ba version

| Version | Adapter path | Declared scope | Train rows | Max sequence | Epochs | Best epoch |
|---|---|---|---:|---:|---:|---:|
| V1 | `V1/all_2757/...best` | `all_2757`; gate gọi `2757_full_target_scope` | **1.930** | 512 | 3 | 2 |
| V2 | `V2/...best` | gate gọi `2757_full_target_scope`; config thiếu experiment label | **1.930** | 768 | 5 | 3 |
| V3 | `V3/all_2757/...best` | `all_2757`; gate gọi `2757_full_target_scope` | **1.930** | 512 | 4 | 3 |

Cả ba `stage03_target_scope_summary.json` đều ghi:

- `target_text_present_cases = 2757`;
- `full_report_cases = 2729`;
- `partial_report_cases = 28`;
- giữ nguyên partial original text;
- không tự sinh clinical label;
- full-reference metrics chỉ dùng full-report cases.

### Kết luận về “bản loại 28 cases”

Không thể xác định có một version riêng biệt đã train 2.729 cases, vì evidence của **cả ba** ghi `train_rows=1930` và scope toàn dataset 2.757. Có thể ý tưởng ablation 2.729-vs-2.757 từng tồn tại trong workflow, nhưng ba adapter đang được evaluation đều khai báo `2757_full_target_scope`; V1/V3 còn có tên rõ `all_2757`.

Vì vậy:

- **V1 = all_2757**;
- **V3 = all_2757**;
- **V2 cũng được adapter audit khai là 2757_full_target_scope**, nhưng provenance kém rõ hơn vì `train_config.json` thiếu `experiment_id` và `target_scope_mode`.
- Không được gán V2 là “2729 model” nếu không có case-ID list hoặc dataloader log chứng minh 28 cases bị loại khỏi loss.

Evidence: [stage03_adapter_provenance_matrix.csv](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_adapter_provenance_matrix.csv).

---

## 4. Vì sao PM Gate không PASS?

### Cấu trúc PM Gate

PM Gate không chỉ hỏi notebook có chạy hay không. Nó là phép AND giữa các điều kiện kỹ thuật, evidence và clinical safety. Một nhánh critical FAIL đủ làm PM cuối FAIL.

```text
Input/data contract: PASS
Technical execution: phần lớn PASS
Evaluation artifacts: PASS
Clinical text safety: FAIL
=> PM Gate: FAIL
=> Clinical readiness: NOT_ESTABLISHED
```

### Lý do trực tiếp

| Chỉ số | V1 | V2 | V3 |
|---|---:|---:|---:|
| Clinical text safety gate | **FAIL** | **FAIL** | **FAIL** |
| Critical-error-any | 372/414 | 374/414 | 373/414 |
| Critical-error-any rate | 89,86% | 90,34% | 90,10% |

Tỷ lệ gần 90% nghĩa là rule-based screening tìm thấy ít nhất một lỗi critical ở gần chín trên mười reports. Dù một số rules bị parser ảnh hưởng, đây vẫn vượt xa mức có thể chấp nhận.

### Tại sao technical execution PASS không cứu được PM Gate?

- Notebook chạy hết chỉ chứng minh pipeline hoàn thành.
- Data contract PASS chỉ chứng minh đúng cohort/case IDs.
- BLEU/BERTScore tính được chỉ chứng minh metric có output.
- Không điều nào chứng minh report đúng ảnh hoặc không gây hại.
- Safety gate là hard veto; do đó PM phải FAIL.

### Mâu thuẫn adapter gate

Cả ba `stage03_pm_gate.json` ghi đại ý:

```text
adapter_saved.pass = true
mistral_lora_adapter = false
mistral_lora_adapter_best = false
```

Đây là lỗi logic/evidence semantics. Evaluation sử dụng adapter từ Kaggle input nên output evaluation không nhất thiết chứa bản sao adapter. Gate đúng phải ghi `EXTERNAL_ADAPTER_PROVENANCE_VERIFIED` hoặc `NOT_APPLICABLE`, thay vì PASS khi hai boolean existence đều false.

### Duplicate/leakage chưa được loại

Stage02 ghi:

- 3 nhóm duplicate report cross-split, 8 cases;
- 17 near-duplicate pairs ≥0,95;
- 8 cross-split near-duplicate pairs;
- trạng thái `QUANTIFIED_NOT_REMOVED`.

Điều này không phải nguyên nhân trực tiếp duy nhất làm PM FAIL, nhưng làm text similarity có nguy cơ lạc quan và hạn chế khả năng tuyên bố generalization.

---

## 5. Giải thích toàn bộ nhóm metric thấp

## 5.1 BLEU-4

| V1 | V2 | V3 |
|---:|---:|---:|
| 0,0582 | **0,0838** | 0,0586 |

BLEU-4 đo mức trùng các chuỗi bốn từ. Mức 0,058–0,084 rất thấp, cho thấy model hiếm khi giữ đúng cụm từ dài và cấu trúc giống reference.

**Vì sao thấp:**

- output chỉ dài khoảng 44–45% reference;
- phần cuối report thường bị cắt;
- model đổi cơ quan, vị trí, kích thước và SUV;
- câu bị loạn chính tả nên n-gram không khớp;
- model sinh template khác reference.

BLEU thấp không tự động nghĩa là sai y tế vì có nhiều cách diễn đạt đúng. Nhưng kết hợp với omission/hallucination và raw outputs loạn nghĩa, nó phản ánh lỗi thật chứ không chỉ khác văn phong.

## 5.2 ROUGE-L F1

| V1 | V2 | V3 |
|---:|---:|---:|
| 0,3614 | **0,3926** | 0,3627 |

ROUGE-L đo chuỗi con dài nhất, nhạy với độ bao phủ và thứ tự nội dung. Mức 0,36–0,39 là thấp–trung bình.

**V2 cao hơn** vì sinh nhiều template PET/CT, FDG, kích thước và SUV hơn. Nhưng một câu có cấu trúc giống reference vẫn có thể chứa **vị trí/SUV bịa**. Do đó ROUGE cao hơn không đồng nghĩa an toàn hơn.

Paired bootstrap xác nhận V2 cao hơn V1 `Δ +0,0588`, CI95% `[+0,0555; +0,0622]`; đây là khác biệt thật về text overlap, không phải bằng chứng clinical correctness.

## 5.3 BERTScore F1

| V1 | V2 | V3 |
|---:|---:|---:|
| 0,7255 | **0,7604** | 0,7305 |

BERTScore đo gần nghĩa embedding ở token-level. Điểm 0,73–0,76 có vẻ cao hơn BLEU vì nhiều thuật ngữ chung như “FDG”, “tăng chuyển hóa”, “hạch”, “không phát hiện”.

**Giới hạn nghiêm trọng:** một report nói sai cơ quan hoặc đảo phủ định vẫn có thể dùng cùng từ y khoa và đạt BERTScore tương đối cao. Vì vậy V2 cao nhất không bù được 295 abnormal-to-normal và 44 SUV hallucinations.

## 5.4 Keyword metrics

### Bảng tổng hợp Keyword/Entity
| Metric | V1 | V2 | V3 |
|---|---:|---:|---:|
| Keyword Precision | **0,9856** | 0,8479 | 0,9832 |
| Keyword Recall | **0,4236** | 0,3947 | 0,4023 |
| Keyword F1 | **0,5916** | 0,5357 | 0,5695 |

### Bảng phân rã đối chiếu chi tiết Recall
Để làm rõ mức độ bỏ sót, chúng ta cần đối chiếu khả năng mô hình thu hồi "từ vựng chung" (ROUGE-L Recall) và khả năng thu hồi "thực thể y tế cốt lõi" (Keyword Recall):

| Phân loại Recall (Thu hồi) | V1 | V2 | V3 | Ý nghĩa thực tế |
|---|---:|---:|---:|---|
| **Textual (ROUGE-L)** | 0,3614 | **0,3926** | 0,3627 | Chỉ 36-39% chuỗi từ vựng của bác sĩ được mô hình giữ lại. |
| **Medical Keyword** | **0,4236** | 0,3947 | 0,4023 | Chỉ 39-42% các từ khóa bệnh lý, giải phẫu cốt lõi được mô hình sinh ra. |

- **Precision (Độ chuẩn xác) rất cao ở V1/V3 (0,98):** Tức là, mỗi khi mô hình V1 hoặc V3 quyết định viết ra một từ khóa (keyword) y tế nào đó, thì 98% khả năng từ khóa đó là đúng (có tồn tại trong báo cáo gốc của bác sĩ).
- **Recall (Độ bao phủ/Độ thu hồi) cực kỳ thấp (0,39–0,42):** Đây là điểm "chết người". Recall chỉ 0,42 nghĩa là trong tổng số 100 từ khóa quan trọng (tên cơ quan, bệnh lý, triệu chứng) mà bác sĩ đã viết trong báo cáo gốc, mô hình **chỉ nhắc đến được 42 từ khóa, và bỏ sót (quên) tới 58 từ khóa**. 
- **V2 có Precision thấp hơn (0,84):** Điều này chứng tỏ V2 có xu hướng "bịa" (hallucination) thêm nhiều từ khóa hoặc bệnh lý mà báo cáo gốc của bác sĩ không hề đề cập.

**Giải thích sâu về tác hại của Recall thấp trong Y tế (Omission Error):**
Việc Recall thấp ở mức 0,42 là dấu hiệu của lỗi **Omission (Bỏ sót thông tin)** rất nặng nề. Trong báo cáo PET/CT, nếu mô hình bỏ sót các từ khóa chỉ cơ quan (ví dụ: hạch, lách, màng phổi) hoặc trạng thái bệnh (ví dụ: di căn, ác tính), bệnh nhân có thể bị chẩn đoán sót bệnh. Mức độ Recall này hoàn toàn không đủ tiêu chuẩn để dùng làm trợ lý y khoa.

**Nguyên nhân gốc rễ khiến Recall thấp:** 
Sự kết hợp giữa **Recall thấp** và **Precision cao** (như ở V1) là hệ quả trực tiếp của lỗi **Truncation (Cắt cụt)** đã phân tích ở Mục 5.13. Vì mô hình chỉ được phép sinh tối đa 512 hoặc 768 tokens (không đủ chỗ để viết hết báo cáo), nó chỉ viết được khoảng 45% đầu tiên của báo cáo. Những gì nó viết ra thì rất đúng (Precision cao), nhưng do bị "hết giấy giữa chừng" (cắt cụt), nó không thể viết tiếp 55% nội dung còn lại, dẫn đến bỏ sót hàng loạt từ khóa quan trọng ở nửa sau báo cáo và toàn bộ phần NHẬN ĐỊNH (Recall thấp).

## 5.5 Organ F1

| V1 | V2 | V3 |
|---:|---:|---:|
| **0,6915** | 0,5909 | 0,6756 |

Organ F1 đo model có nhắc đúng các nhóm cơ quan/vùng giải phẫu hay không. V2 thấp nhất vì raw outputs thường di chuyển phát hiện giữa não, phổi, gan, hạch và bụng-chậu. V1 tốt nhất nhưng 0,69 vẫn đồng nghĩa độ bao phủ/độ chính xác cơ quan chưa đầy đủ.

## 5.6 Finding-status F1

| V1 | V2 | V3 |
|---:|---:|---:|
| **0,7020** | 0,6866 | 0,6901 |

Metric này xem entity/finding có trạng thái positive/negative phù hợp không. Mức ~0,69–0,70 vẫn để lại khoảng sai số lớn, đặc biệt nguy hiểm nếu đổi “tăng FDG” thành “không tăng” hoặc ngược lại.

Không được hiểu F1 0,70 là “70% clinically correct”; metric chỉ bao phủ vocabulary/rule được thiết kế.

## 5.7 Abnormal-to-normal

| V1 | V2 | V3 |
|---:|---:|---:|
| 33 | **295** | **26** |

Đây là failure mode nguy hiểm: reference bất thường nhưng prediction chuyển thành câu bình thường/không phát hiện.

- V2 cực xấu: 71,26% cases bị flag.
- V3 thấp nhất, V1 gần V3.
- Cần physician adjudication vì rule có thể false positive; nhưng chênh lệch V2 quá lớn và raw examples xác nhận V2 có phủ định/đổi nội dung.

## 5.8 Missed malignancy

Cả ba đều `209/414 = 50,48%`.

Con số giống hệt nhau là dấu hiệu rule phụ thuộc mạnh vào reference label/section parser, không phản ánh phân biệt version tốt. Không được coi 209 là 209 lỗi đã được bác sĩ xác nhận. Tuy nhiên raw outputs cho thấy model thực sự bỏ cụm malignancy/metastases và phần Impression do truncation.

## 5.9 SUV hallucination và SUV omission

| Metric | V1 | V2 | V3 |
|---|---:|---:|---:|
| SUV hallucinated | 0 | **44** | 3 |
| SUV missed | **360** | 0 | **352** |

Hai failure modes đối nghịch:

- V1/V3 gần như không sinh giá trị SUV → ít hallucination nhưng omission rất cao.
- V2 sinh SUV rộng rãi → không bị missed theo rule, nhưng 44 cases có SUV không được reference hỗ trợ.

V2 không “tốt về SUV”; nó chỉ đổi omission thành hallucination. Trong y tế, một SUV bịa có thể làm thay đổi phân tầng nguy cơ, response assessment và vị trí sinh thiết.

## 5.10 FDG missed = 0

FDG missed bằng 0 không chứng minh model giữ đúng FDG finding. Raw outputs dùng từ `FDG` rất thường xuyên theo template, trong khi cơ quan, trạng thái và SUV có thể sai. Metric mention-level này quá dễ đạt.

## 5.11 Clean Vietnamese và contamination

| V1 | V2 | V3 |
|---|---:|---:|---:|
| 412/414 — 99,52% | 381/414 — 92,03% | 410/414 — 99,03% |

V2 xấu nhất với 33 cases không đạt clean-Vietnamese screening. Tuy nhiên V1/V3 có rate cao **không có nghĩa tiếng Việt tốt**. Detector chỉ bắt English/CJK/foreign script; nó bỏ sót nhiều lỗi như:

- `MÔ TÃ HÌNH ÁNH`;
- `Hình đạo`, `Hình dạy`, `khôn khéo`;
- `tăng chú trọng`, `chuyển hoại`, `bắt xãt`;
- câu đúng chữ Việt nhưng vô nghĩa y khoa.

Do đó `gibberish_rate=0` là false reassurance: rule gibberish hiện tại không nhạy với loạn tiếng Việt nội ngôn ngữ.

## 5.12 Prompt leak và CJK = 0

Đây là điểm tốt thật về mặt kỹ thuật: model không lặp prompt và không sinh CJK theo detector. Nhưng không liên quan trực tiếp đến clinical correctness và không bù omission/hallucination.

## 5.13 Short reports và truncation

| Metric | V1 | V2 | V3 |
|---|---:|---:|---:|
| Mean prediction words | 169,0 | 173,8 | 168,6 |
| Mean reference words | 385,8 | 385,8 | 385,8 |
| Short report | 98,78% | 93,89% | 100% |
| Likely truncated | 93,00% | 95,89% | 94,44% |

Prediction chỉ khoảng 44–45% độ dài reference. Phần Impression thường biến mất. Đây là nguyên nhân nền của keyword recall thấp, malignancy omission và SUV omission.

### Gốc training-side

Cả ba config ghi `train_truncation_rate = 1.0` và `eval_truncation_rate = 1.0` ở token-budget audit:

- V1/V3 max sequence 512;
- V2 max sequence 768;
- total token p50 khoảng 1.583–1.621;
- p95 gần 1.944–1.954.

Nghĩa là **100% mẫu audit dài hơn sequence budget**. Model được học trên targets bị cắt, nên học report không đầy đủ và heading cuối bị mất. Đây là lỗi thiết kế training quan trọng nhất.

> [!WARNING]
> **Đính chính sự hiểu lầm cực kỳ nguy hiểm về thuật ngữ trong cấu hình:**
> Trong file cấu hình, con số ghi là `"truncation_rate_at_max_seq": 1.0`. Ở đây, `truncation_rate` có nghĩa là **Tỷ lệ bị cắt cụt** (bỏ đuôi). Và `1.0` trong thống kê mang ý nghĩa là **100%**. Nghĩa là báo cáo đang thừa nhận: **100% toàn bộ dữ liệu đưa vào huấn luyện ĐỀU BỊ CẮT CỤT**. Nó KHÔNG PHẢI là "bao phủ đủ 100%".
>
> Nhìn kỹ vào các con số thực tế mà file audit đã ghi lại cho tập huấn luyện (train):
> - **Đầu vào (prompt):** Trung bình cần 455 tokens.
> - **Đầu ra (target/báo cáo):** Trung bình cần 1.127 tokens.
> - **Tổng cộng (total):** Trung bình một ca cần khoảng 1.583 tokens (ở p50) và những ca dài cần tới 1.953 tokens (ở p95).
>
> Tuy nhiên, giới hạn huấn luyện (`max_seq_length`) lại chỉ được đặt là 768 (đối với V2) và 512 (đối với V1, V3).
>
> **Hệ quả:** Khi tổng số token thực tế cần là 1.583, nhưng cửa sổ học chỉ chứa được 768 token, mô hình sẽ tự động lấy dao "chặt đứt" phần đuôi kể từ token số 769 trở đi. Vì toàn bộ các ca (cases) đều dài hơn 768, nên 100% số ca đã bị chặt mất phần cuối. Phần bị chặt mất này không đâu khác chính là toàn bộ đoạn `NHẬN ĐỊNH KẾT QUẢ` (Impression) và một phần đuôi của đoạn `MÔ TẢ HÌNH ẢNH`.
>
> Đó là lý do giải thích hoàn hảo cho việc:
> 1. Tại sao mô hình không sinh ra đoạn `NHẬN ĐỊNH` khi test (vì lúc đi học, sách giáo khoa của nó đã bị xé mất trang cuối).
> 2. Tại sao sinh ra chữ bị loạn và ngắn hơn thực tế.

## 5.14 Training/validation loss

| Version | Epochs | Best epoch | Best val loss |
|---|---:|---:|---:|
| V1 | 3 | 2 | 0,000714 |
| V2 | 5 | 3 | 0,139084 |
| V3 | 4 | 3 | 0,000548 |

Không được kết luận V3 tốt hơn V2 hàng trăm lần chỉ từ val loss. V1/V3 loss cực thấp bất thường trong khi generation quality rất xấu, gợi ý:

- loss mask/collator hoặc denominator khác;
- nhiều padding/ignored tokens;
- target truncation làm task ngắn và dễ;
- metric loss không tương thích trực tiếp giữa runs;
- possible template memorization.

Validation loss không dự báo clinical safety trong kết quả này.

---

## 6. Lỗi parser làm metric nào không đáng tin?

Parser trạng thái:

| Status | V1 | V2 | V3 |
|---|---:|---:|---:|
| Prediction unparsed | 414/414 | 414/414 | 414/414 |
| Reference findings-only | 413/414 | 413/414 | 413/414 |
| Impression eligible | 1/414 | 1/414 | 1/414 |

Raw reference có `NHẬN ĐỊNH` khoảng 410/414 reports. Prediction thường sinh heading sai `MÔ TÃ HÌNH ÁNH`. Parser exact/fuzzy hiện tại không nhận các heading này.

### Metric phải tạm loại khỏi lựa chọn chính

- Findings keyword recall hiện báo 0 với n=413;
- Impression keyword recall 0 với n=1;
- Findings–Impression contradiction bằng 0;
- section completeness/section ROUGE;
- bất kỳ gate nào dựa trên section denominator hiện tại.

Không được nói model “không mâu thuẫn Findings–Impression”; parser không tìm thấy sections để so sánh.

---

## 7. Chín ví dụ raw output

Evidence nguyên văn đầy đủ: [stage03_9_raw_output_examples.csv](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_9_raw_output_examples.csv).

> [!WARNING]
> Các ví dụ dưới đây cho thấy cả ba version đều sinh tiếng Việt có vẻ giống report nhưng thường **vô nghĩa y khoa**, đổi cơ quan và bị cắt. Detector “clean Vietnamese” bỏ sót phần lớn vấn đề này.

### V1 — ví dụ 1: loạn ngôn ngữ nặng

Case `2968b594e0db`:

```text
Tăng nhẹ chuyển hoá FDG ... theo đạo chỉ địa điểм.
Hình đạo khu trú ở nhu mô não vùng thái dương bên trái.
Hình ảnh vài hạch nhỏ, khôn khéo trong vùng thúy thái bên tráo.
```

- Có ký tự Cyrillic `м` trong `điểм`.
- Các cụm “đạo chỉ địa”, “hạch nhỏ, khôn khéo”, “thúy thái” vô nghĩa.
- Reference nói hạch cổ, phổi, gan, phúc mạc và metastases; prediction chuyển trọng tâm thành tổn thương não không được hỗ trợ.
- Flag: missed malignancy, SUV missed, truncation.
- Mức nguy cơ: **Critical**.

### V1 — ví dụ 2: bỏ tổn thương ác tính

Case `12cfd770b8bf`:

```text
Hình đạo giảm tỷ trọng và giảm chuyển trong nhu mô não.
Hình ánh bắt tăng chuyển nhu mô FDG ấm ức ở não.
NHỆT ĐỎI KẠT QUỔA...
```

Reference có u lympho ác tính hốc mũi xâm lấn hốc mắt/xoang, SUVmax 12,6. Prediction không giữ tổn thương chính và Impression bị loạn hoàn toàn.

- Abnormal-to-normal.
- Missed malignancy.
- SUV missed.
- Mức nguy cơ: **Critical**.

### V1 — ví dụ 3: “tốt hơn” tương đối nhưng vẫn không đạt

Case `c1f034a8c785`:

```text
Hình đạo giảm tỷ trọng và giảm chuyển trong amydal phải bên trái so với bên trăo.
Hình ảnh vài hạch nhỏ ... bên tráis trong rì a lưỵi.
```

Metric entity tương đối cao, nhưng câu vẫn vô nghĩa, đổi giải phẫu và output bị cắt trước Impression. Đây là bằng chứng ngay cả best-looking case của V1 chưa đủ chất lượng lâm sàng.

### V2 — ví dụ 1: loạn tiếng Việt và SUV bịa

Case `c2a625c59a45`:

```text
MÔ TÃ T HÌNH ÁNH
... theo đạc điêm sinh lý.
... tăng chyển hóа FDG
... khô ngời, khôn khéo, khong tăng chuyn hoá FDG.
```

- Có Cyrillic `а` trong `hóа`.
- Nhiều lỗi tách chữ và chính tả.
- Reference SUV vòm họng 30,3 và hạch 28,7/23,4/18; prediction đổi thành 11, 3,5, 3 và thêm tổn thương phổi SUV 4,5.
- Đây là cả numeric degradation lẫn unsupported finding.
- Mức nguy cơ: **Critical**.

### V2 — ví dụ 2: safety worst

Case `7084f604c7c5`:

```text
Hình ẩn nhỏ nhóm IIA bên phảihai kích thủy 10 mm...
Không phát hiệnh hình tràn dịch màng phổi hai bén...
... tăng chuyn hoại FGD...
```

Reference có lymphoma lan rộng nhiều vùng hạch và lách. Prediction sinh tổn thương vòm họng/thực quản khác reference, đảo trạng thái và bị cắt.

- Abnormal-to-normal.
- Missed malignancy.
- Negation flip.
- Truncation.
- Mức nguy cơ: **Critical**.

### V2 — ví dụ 3: finding-status F1 = 1 nhưng nội dung vẫn sai

Case `ded4c601c029`:

```text
Hình đám ở thành sau hạ họng và dây thanh âm bên phải...
... thùy dưới phổi trài ... SUVmax: 10,1.
```

Reference nói khối thùy trên phổi phải SUVmax 7,8 và phổi trái sau mổ/xơ hóa. Prediction đổi sang hạ họng/dây thanh, đổi bên/vị trí phổi và SUV 10,1. Finding-status F1 bằng 1 chỉ phản ánh các trạng thái được rule bắt, không chứng minh nội dung đúng.

### V3 — ví dụ 1: loạn nghĩa và SUV hallucination

Case `ce069a34f494`:

```text
Hình đạo dày ... tại nhu mô não vùng chẩn đoạt trái.
Tăng chuyễn hóa tỷ số FDG từ 3,5 lần ... tại điểм đặc.
Hình khoang bụng tăng tỷ tỷ, kích tỷ thứ hai.
```

Reference chủ yếu là amydal đối xứng và không có SUV numeric. Prediction thêm tổn thương não/SUVmax 11 và các cụm vô nghĩa.

- Abnormal-to-normal flag.
- SUV hallucination.
- Truncation.
- Mức nguy cơ: **Critical**.

### V3 — ví dụ 2: bỏ ung thư phổi di căn

Case `365af1db7ebf`:

```text
Hình đạo giảm tỷ trọng ... vùng trán đỉnh...
Ngực: Hình dải khuyết xạ nhu mô não.
Không phát hiện đười tăng chú trọng FDG.
```

Reference có khối phổi trái 77×36 mm SUVmax 25, hạch, gan, thượng thận, phúc mạc và di căn xương. Prediction chuyển sang não, phủ định bất thường và bỏ gần toàn bộ disease burden.

- Abnormal-to-normal.
- Missed malignancy.
- SUV missed.
- Truncation.
- Mức nguy cơ: **Critical**.

### V3 — ví dụ 3: entity metric tốt nhưng văn bản vẫn loạn

Case `fef01acad726`:

```text
Hình đạo giảm tỷ trọng ... tại thùy thái phải túy giáp...
Hình ảnh vài hạch nhỏ tăng chuẩn đoán rìa lúc bắt xãt.
Hình dạ đối xứng, khôn khéo.
```

Dù keyword F1 0,706, organ F1 0,783 và finding-status F1 0,889, report vẫn vô nghĩa, thêm findings không rõ căn cứ và bị cắt. Đây là ví dụ rõ rằng metric entity tốt hơn không đồng nghĩa report dùng được.

---

## 8. Ma trận chọn version

| Tiêu chí | V1 | V2 | V3 | Người thắng tương đối |
|---|---|---|---|---|
| BLEU/ROUGE/BERTScore | Thấp hơn V2 | **Cao nhất** | Gần V1 | V2 |
| Keyword coverage | **Tốt nhất** | Tệ nhất | Thứ hai | V1 |
| Organ F1 | **Tốt nhất** | Tệ nhất | Thứ hai | V1 |
| Finding-status F1 | **Tốt nhất** | Thứ ba | Thứ hai | V1 |
| Clean-Vietnamese screening | **Tốt nhất** | Tệ nhất | Thứ hai | V1 |
| Abnormal-to-normal | 33 | **295 — rất xấu** | **26 — thấp nhất** | V3 |
| SUV hallucination | **0** | 44 | 3 | V1 |
| SUV omission | 360 | **0** | 352 | V2, nhưng đổi thành hallucination |
| Truncation | 93,00% | 95,89% | 94,44% | V1, nhưng đều rất xấu |
| PM/safety gate | FAIL | FAIL | FAIL | Không ai |
| Raw-text coherence | Rất xấu | Rất xấu; nhiều lỗi nhất | Rất xấu | Không ai |

### Thứ tự đề xuất để tiếp tục development

1. **V1** — ưu tiên sửa tiếp vì entity/organ/language tốt nhất và không SUV hallucination.
2. **V3** — dự phòng, abnormal-to-normal thấp hơn V1 nhưng entity metrics thấp hơn và raw text vẫn loạn.
3. **V2** — không ưu tiên safety-first dù text similarity cao; 295 abnormal-to-normal, 44 SUV hallucination và 33 language failures là hard concerns.

### Khi nào có thể đảo lựa chọn?

- Nếu sau sửa parser, physician review chứng minh abnormal-to-normal rule false-positive chủ yếu ở V2, cần đánh giá lại.
- Nếu V1 không khắc phục được SUV omission sau retraining, V3 hoặc một checkpoint khác có thể tốt hơn.
- Nếu tìm được adapter thực sự train chỉ trên 2.729 full-report cases, phải đánh giá lại đúng adapter đó trên cùng 414-case cohort; ba adapter hiện tại không cung cấp bằng chứng đó.

---

## 9. Toàn bộ lỗi cần sửa theo tầng

### Dữ liệu

- 28 partial reports: 27 thiếu Impression, 1 thiếu Findings.
- Duplicate/near-duplicate cross-split chưa loại.
- SUV outlier 76 cần expert review.
- Thiếu blinded label/adjudication.

### Training

- 100% token-budget truncation.
- Max sequence 512/768 thấp hơn total token p50 ~1.600.
- Loss không phản ánh generation quality.
- Có thể học template report thay vì mapping image-to-finding.
- 2.5D visual summary không chứng minh đọc đủ PET/CT volume.

### Generation

- 93–96% likely truncated.
- Heading sai chính tả.
- Phần Impression thường mất.
- Loạn tiếng Việt nội ngôn ngữ detector không bắt.
- Đổi cơ quan/bên/vị trí.
- Omission malignancy và SUV.
- Hallucination SUV/findings, đặc biệt V2.
- Câu phủ định/khẳng định không ổn định.

### Evaluation

- Parser fail 100% predictions.
- Impression denominator 1/414 sai nghiêm trọng.
- Section metrics/contradiction không đáng tin.
- Missed-malignancy giống hệt 209 ở ba version cần kiểm tra rule.
- Adapter gate tự mâu thuẫn.
- `gibberish=0` không bắt loạn tiếng Việt semantic.
- Text similarity không image-grounded.

---

## 10. Hành động bắt buộc trước vòng chọn model tiếp theo

1. Sửa parser headings và tái tính metric từ outputs hiện có; không cần generation lại.
2. Thêm Vietnamese semantic/orthographic quality detector, không chỉ foreign-script detector.
3. Tăng context/target strategy để không truncate 100% training samples.
4. Thiết kế loss/generation bảo đảm đủ Findings và Impression.
5. Tách SUV mention, numeric match, unsupported numeric và eligible denominator.
6. Review case-level abnormal-to-normal/malignancy bằng ít nhất hai bác sĩ y học hạt nhân.
7. Chạy image-grounded review với PET/CT đầy đủ.
8. Đánh giá external cohort trước khi tuyên bố generalization.
9. Chỉ sau các bước trên mới chọn checkpoint/model cho clinical development.

---

## 11. Lộ trình khắc phục cụ thể

Từ những nhận định trên, dưới đây là lộ trình khắc phục cụ thể được chia thành 4 giai đoạn, từ dễ đến khó, để giải quyết tận gốc vấn đề:

### 1. Khắc phục ngay lập tức: Sửa công cụ đánh giá (Evaluation Harness)
Trước khi train lại bất kỳ mô hình nào, ta phải sửa thước đo để nó đo đúng.
- **Vấn đề:** Trình phân tích cú pháp (parser) hiện tại tìm kiếm chính xác chữ `MÔ TẢ HÌNH ẢNH` và `NHẬN ĐỊNH KẾT QUẢ`. Khi mô hình viết sai chính tả thành `MÔ TÃ HÌNH ÁNH` hoặc `NHỆT ĐỎI KẠT QUỔA`, parser không nhận diện được, dẫn đến 100% đánh giá phần section bị fail.
- **Cách khắc phục:**
  - **Cập nhật Regex/Fuzzy Matching:** Viết lại hàm parser trong notebook đánh giá để cho phép sai số chính tả có kiểm soát (ví dụ: dùng thư viện `fuzzywuzzy` hoặc regex linh hoạt để bắt các cụm có độ tương đồng > 80% so với "MÔ TẢ HÌNH ẢNH").
  - **Đo lại (Re-evaluate):** Chạy lại metric trên các file `generated_reports.csv` hiện có (không cần mất tiền/thời gian chạy GPU sinh lại text) để xem khi nhận diện được tiêu đề, điểm Impression và Findings thực sự của mô hình là bao nhiêu.

### 2. Khắc phục khâu Huấn luyện (Training) để chống Truncation
Đây là nguyên nhân cốt lõi khiến câu bị cắt cụt, mất phần Impression và làm các chỉ số keyword thấp thảm hại.
- **Vấn đề:** Độ dài token thực tế của báo cáo trung bình là ~1.600 token. Nhưng `max_seq_length` trong cấu hình (V1/V3 là 512, V2 là 768) quá nhỏ. Mô hình bị "ép" học các văn bản cụt lủn và không bao giờ học được cách kết thúc một báo cáo y tế một cách hoàn chỉnh.
- **Cách khắc phục:**
  - **Tăng `max_seq_length`:** Bắt buộc phải nâng lên tối thiểu **2048** trong code Kaggle thì mô hình mới học được toàn vẹn 100% nội dung (cover 100%).
  - **Tối ưu VRAM (nếu Kaggle T4/P100 không đủ bộ nhớ):**
    - Bật tính năng **Gradient Checkpointing** (để tiết kiệm VRAM dù sequence dài).
    - Giảm `batch_size` xuống 1 hoặc 2, và tăng `gradient_accumulation_steps` lên tương ứng (ví dụ 16 hoặc 32) để duy trì effective batch size.
    - Sử dụng P100 (16GB VRAM) hoặc lý tưởng nhất là 2x T4 (nếu Kaggle hỗ trợ) với chiến lược chia model.
  - **Đảm bảo EOS token:** Kiểm tra kỹ quá trình tiền xử lý (tokenize) để chắc chắn token kết thúc `</s>` (EOS) luôn được đính kèm ở cuối phần `NHẬN ĐỊNH`, buộc mô hình học cách dừng sinh chữ khi đã báo cáo xong.

### 3. Khắc phục khâu Dữ liệu (Data/Prompting) để chống Loạn ngôn ngữ & Bịa đặt (Hallucination)
Mô hình (đặc biệt V2) đang sinh ra chữ tiếng Việt bị loạn (chữ Cyrillic) và bịa chỉ số SUV.
- **Vấn đề:** Base model Mistral-7B bản gốc không phải là mô hình chuyên tiếng Việt. Dưới áp lực ép học từ chuyên ngành y tế bằng tiếng Việt với sequence bị cắt cụt, nó bị "vỡ" ngôn ngữ.
- **Cách khắc phục:**
  - **Thêm System Prompt định hướng mạnh:** Thay đổi cấu trúc prompt khi train. Bắt buộc thêm một câu lệnh nghiêm ngặt: *"Bạn là bác sĩ Y học hạt nhân người Việt Nam. Phải viết báo cáo bằng tiếng Việt chuẩn xác, đúng chính tả, không trộn ngôn ngữ lạ. Phải trích xuất chính xác chỉ số SUV, tuyệt đối không bịa đặt số liệu."*
  - **Lọc dữ liệu rác (Data Cleaning):** Xem lại file chứa 1.930 cases train. Có thể có một số cases bị nhiễu dấu câu hoặc encoding lỗi.
  - **Chuyển base model (Cân nhắc dài hạn):** Thay vì dùng Mistral-7B mặc định, có thể thử dùng một base model đã được pre-train tốt tiếng Việt (ví dụ: VinaLLaMA, PhoGPT, hoặc Mistral phiên bản đã fine-tune tiếng Việt).

### 4. Khắc phục khâu Đánh giá Lâm sàng (Clinical Safety)
- **Vấn đề:** Lỗi "Abnormal-to-normal" (biến bệnh ác tính thành bình thường) quá cao (V2 lên tới 295 cases).
- **Cách khắc phục:**
  - **Áp dụng "Loss Masking" (Trọng số phạt):** Trong quá trình tính Loss lúc training, hãy thiết kế để hàm loss phạt cực nặng đối với các token liên quan đến từ khóa "tăng chuyển hóa", "ác tính", "di căn", "SUVmax". Nếu mô hình đoán sai các từ khóa này, loss sẽ tăng vọt, ép mô hình phải học thuộc lòng các chỉ số sinh tồn của bệnh nhân thay vì chỉ học cấu trúc câu.
  - **Trích xuất SUV có kiểm soát:** Nếu mô hình ngôn ngữ sinh số quá kém, cân nhắc đổi bài toán. Đừng bắt mô hình sinh cả đoạn văn dài. Hãy train mô hình sinh ra một file JSON có cấu trúc (ví dụ: `{"Cơ quan": "Phổi", "Tình trạng": "Tăng chuyển hóa", "SUVmax": 10.1}`). Sau đó dùng rule/code để ghép JSON thành đoạn văn báo cáo. Cách này an toàn y tế hơn rất nhiều.

## Evidence

- [Báo cáo audit trước](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_3version_medical_audit_report.md)
- [Bảng metric tổng hợp](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_3version_summary.csv)
- [Paired bootstrap](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_3version_paired_bootstrap.csv)
- [Parser diagnostics](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_parser_diagnostics.csv)
- [Adapter provenance](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_adapter_provenance_matrix.csv)
- [Toàn bộ metric/gate JSON keys](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_all_json_metric_gate_keys.csv)
- [Nguồn JSON gate/metric hợp nhất](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_complete_metric_gate_source.json)
- [9 raw examples](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_9_raw_output_examples.csv)
- [Inventory 356 files](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/mistral7b_complete_file_inventory.csv)
