# Báo cáo kiểm toán Stage03 V1–V3

## Kết luận ngắn gọn

> [!CAUTION]
> **Không version nào đạt yêu cầu an toàn văn bản lâm sàng.** Cả V1, V2 và V3 đều có `PM Gate = FAIL`, `clinical readiness = NOT_ESTABLISHED`.

- Ba version được đánh giá công bằng trên cùng **414/414 cases**, cùng cohort hash và reference report.
- V2 có điểm tương đồng văn bản cao nhất, nhưng đồng thời có dấu hiệu nguy hiểm nhất ở một số trục: **295/414 abnormal-to-normal flags**, **44 SUV hallucination flags**, 33 cases không đạt clean-Vietnamese screening.
- V1 có keyword/organ metrics tốt nhất và ít abnormal-to-normal hơn V2, nhưng bỏ sót SUV ở **360/414** cases theo screening.
- V3 cân bằng hơn V1/V2 ở một số metric, nhưng vẫn có **352/414 SUV-missed flags**, 100% short reports và PM Gate FAIL.
- Safety screening đánh dấu ít nhất một critical error ở **372–374/414 cases** cho cả ba version. Đây là mức không thể chấp nhận để sử dụng lâm sàng.
- Tuy nhiên, parser section của harness cũng thất bại nghiêm trọng: **414/414 predictions unparsed** và chỉ **1/414 references** được nhận diện Impression. Vì vậy các metric Impression, Findings–Impression contradiction và một phần critical-rule counts phải được xem là **không đủ độ tin cậy**, không phải bằng chứng lâm sàng đã xác nhận.

## 1. Tính công bằng và khả năng tái lập

| Kiểm tra | V1 | V2 | V3 | Kết luận |
|---|---:|---:|---:|---|
| Cases đánh giá | 414 | 414 | 414 | Đủ |
| Unique case IDs | 414 | 414 | 414 | Không duplicate ID |
| Cohort SHA256 | cùng hash | cùng hash | cùng hash | So sánh paired hợp lệ |
| Reference reports | cùng dữ liệu | cùng dữ liệu | cùng dữ liệu | Công bằng |
| Code cells chạy | 21/21 | 21/21 | 21/21 | Hoàn tất |
| Notebook error output | 0 | 0 | 0 | Không traceback |
| Logic sau chuẩn hóa version/path | giống nhau | giống nhau | giống nhau | Chỉ khác adapter/version |

## 2. Bảng so sánh tổng hợp — không che metric yếu

### 2.1 Safety và chất lượng cấu trúc

| Metric | V1 | V2 | V3 | Diễn giải |
|---|---:|---:|---:|---|
| PM Gate | **FAIL** | **FAIL** | **FAIL** | Không version nào qua gate |
| Clinical readiness | NOT ESTABLISHED | NOT ESTABLISHED | NOT ESTABLISHED | Không được dùng lâm sàng |
| Critical error-any screening | **372/414 (89,86%)** | **374/414 (90,34%)** | **373/414 (90,10%)** | Rất xấu; cần bác sĩ xác nhận |
| Abnormal → normal screening | 33/414 (7,97%) | **295/414 (71,26%)** | 26/414 (6,28%) | V2 đặc biệt đáng lo |
| Missed malignancy screening | **209/414 (50,48%)** | **209/414 (50,48%)** | **209/414 (50,48%)** | Có thể chịu ảnh hưởng rule/parser |
| Negation flip | 0 | 1 | 0 | Rule-based, chưa physician-confirmed |
| SUV hallucinated | 0 | **44** | 3 | V2 tệ nhất |
| SUV missed | **360** | 0 | **352** | V1/V3 gần như không sinh SUV |
| Likely truncated | **385/414 (93,00%)** | **397/414 (95,89%)** | **391/414 (94,44%)** | Cả ba đều rất xấu |
| Short report | **98,78%** | **93,89%** | **100%** | Output chỉ khoảng 44–45% độ dài reference |
| Mean prediction words | 169,0 | 173,8 | 168,6 | Reference trung bình 385,8 từ |
| Không đạt clean-Vietnamese screening | 2 | **33** | 4 | V2 có nguy cơ nhiễu ngôn ngữ/chính tả cao nhất |
| Gibberish flag hiện tại | 0 | 0 | 0 | Không loại trừ lỗi chính tả y khoa |
| Prompt leak | 0 | 0 | 0 | Điểm tốt nhưng không bù được safety failures |

> [!WARNING]
> V2 “không missed SUV” không phải điểm tốt độc lập: V2 sinh SUV ở mọi case và có 44 hallucination flags. V1/V3 lại gần như tránh sinh SUV, dẫn đến missed-SUV cực cao. Đây là hai failure mode đối nghịch.

### 2.2 Metric tương đồng và entity

| Metric | V1 | V2 | V3 | Tốt nhất tương đối |
|---|---:|---:|---:|---|
| BLEU-4 | 0,0582 | **0,0838** | 0,0586 | V2 |
| ROUGE-L F1 | 0,3614 | **0,3926** | 0,3627 | V2 |
| BERTScore F1 | 0,7255 | **0,7604** | 0,7305 | V2 |
| Keyword precision | **0,9856** | 0,8479 | 0,9832 | V1 |
| Keyword recall | **0,4236** | 0,3947 | 0,4023 | V1 |
| Keyword F1 | **0,5916** | 0,5357 | 0,5695 | V1 |
| Organ F1 | **0,6915** | 0,5909 | 0,6756 | V1 |
| Finding-status F1 | **0,7020** | 0,6866 | 0,6901 | V1 |

Các điểm số này chỉ đo agreement với reference text. Chúng **không chứng minh** model đọc đúng PET/CT hoặc an toàn lâm sàng.

## 3. Paired comparison và độ bất định

Bootstrap paired 2.000 lần trên cùng 414 cases cho thấy:

- V2 cao hơn V1 về full ROUGE-L: `Δ +0,0588`, 95% CI `[+0,0555; +0,0622]`.
- Nhưng V2 thấp hơn V1 về keyword recall: `Δ -0,0250`, CI `[-0,0313; -0,0191]`.
- V2 thấp hơn V1 về keyword F1: `Δ -0,0421`, CI `[-0,0495; -0,0351]`.
- V2 thấp hơn V1 về organ F1: `Δ -0,0745`, CI `[-0,0866; -0,0635]`.
- V3 thấp hơn V1 về keyword recall/F1 và organ F1.
- V3 và V1 **chưa đủ bằng chứng khác biệt** về full finding F1: CI `[-0,0032; +0,0184]` đi qua 0.

**Ý nghĩa:** V2 bắt chước bề mặt văn bản tốt hơn, nhưng không tốt hơn về độ bao phủ keyword/cơ quan. Không thể chọn V2 chỉ dựa trên ROUGE/BERTScore.

## 4. Lỗi nghiêm trọng của evaluation harness

### 4.1 Parser section không hoạt động đúng

| Parser status | V1 | V2 | V3 |
|---|---:|---:|---:|
| Prediction `unparsed` | **414/414** | **414/414** | **414/414** |
| Reference `findings_only` | 413/414 | 413/414 | 413/414 |
| Reference `impression_only` | 1/414 | 1/414 | 1/414 |
| Impression eligible | **1/414** | **1/414** | **1/414** |

Nhưng kiểm tra raw reference cho thấy cụm `NHẬN ĐỊNH` xuất hiện khoảng **410/414** reports. Do đó `impression_eligible=1` gần như chắc chắn là lỗi parser/heading segmentation.

Prediction thường sinh heading sai chính tả như:

```text
MÔ TÃ HÌNH ÁNH
MÔ TÃ T HÌNH ÁNH
```

Điều này vừa phản ánh lỗi model, vừa làm parser exact/fuzzy hiện tại bỏ qua toàn bộ prediction.

**Hệ quả:**

- Impression-level metric hiện tại không đủ giá trị để kết luận.
- Internal Findings–Impression contradiction bằng 0 không phải bằng chứng model nhất quán; parser không trích được sections.
- Section completeness rates không thể dùng để xếp hạng chính thức trước khi sửa parser và tái tính.

### 4.2 Mâu thuẫn adapter gate

Cả ba `stage03_pm_gate.json` ghi:

```text
adapter_saved.pass = true
mistral_lora_adapter = false
mistral_lora_adapter_best = false
```

Đây là mâu thuẫn logic gate. Evaluation-only có thể không cần copy adapter vào output, nhưng gate phải ghi `NOT_APPLICABLE_WITH_EXTERNAL_ADAPTER_PROVENANCE`, không được báo PASS trong khi hai bằng chứng tồn tại đều false.

### 4.3 Clinical rules cần physician adjudication

Missed-malignancy bằng đúng `209/414` cho cả ba version là dấu hiệu rule có thể phụ thuộc mạnh vào reference labeling hoặc parser hơn là khác biệt model. Critical counts là **screening flags**, không phải confirmed medical errors.

Tuy nhiên, ngay cả khi có false positives, tỷ lệ output ngắn/truncated >93%, section parser failure 100% và PM Gate FAIL vẫn đủ để bác bỏ clinical readiness hiện tại.

## 5. Đánh giá riêng từng version

### V1

**Điểm tương đối tốt**

- Keyword precision/recall/F1 và organ F1 tốt nhất.
- Chỉ 2 cases không đạt clean-Vietnamese screening.
- Không có SUV hallucination flag.

**Điểm yếu không được che**

- 360/414 SUV-missed flags.
- 385/414 likely truncated.
- 98,78% short reports.
- 372/414 critical-error-any screening.
- Không trích được section từ bất kỳ prediction nào.

### V2

**Điểm tương đối tốt**

- BLEU-4, ROUGE-L và BERTScore cao nhất.
- Finding-status agreement tương đối cao.

**Điểm yếu không được che**

- 295/414 abnormal-to-normal screening flags — cao vượt trội.
- 44 SUV hallucination flags.
- 33 cases không đạt clean-Vietnamese screening.
- Keyword recall/F1 và organ F1 thấp nhất.
- 397/414 likely truncated — tệ nhất.
- 374/414 critical-error-any screening.

### V3

**Điểm tương đối tốt**

- Abnormal-to-normal screening thấp nhất: 26/414.
- Chỉ 3 SUV hallucination flags.
- Text/entity metrics thường nằm giữa V1 và V2.

**Điểm yếu không được che**

- 352/414 SUV-missed flags.
- 391/414 likely truncated.
- 100% short reports.
- 373/414 critical-error-any screening.
- Không có lợi thế có ý nghĩa so với V1 ở full finding F1.

## 6. Kết luận chọn model

> [!IMPORTANT]
> **Không chọn bất kỳ version nào để sử dụng lâm sàng.**

Nếu chỉ chọn ứng viên để **sửa pipeline và nghiên cứu tiếp**:

- **V1** là ứng viên entity/organ coverage tốt nhất, nhưng phải sửa SUV omission và generation length.
- **V2** là ứng viên text-similarity tốt nhất, nhưng failure mode abnormal-to-normal, SUV hallucination và language quality khiến nó không an toàn hơn.
- **V3** có abnormal-to-normal screening thấp nhất và ít SUV hallucination, nhưng SUV omission và truncation vẫn quá nghiêm trọng.

Không có “winner” tổng thể vì ưu thế của mỗi version nằm ở metric khác nhau và tất cả đều vi phạm safety gate.

## 7. Bước bắt buộc trước khi đánh giá lại

1. Sửa parser để nhận đúng `MÔ TẢ HÌNH ẢNH`, `NHẬN ĐỊNH KẾT QUẢ`, `NHẬN ĐỊNH`, `KẾT LUẬN` và các heading lỗi chính tả có kiểm soát.
2. Tái tính toàn bộ section/Impression metrics và contradiction flags.
3. Điều tra vì sao 93–96% output bị likely-truncated; kiểm tra token cap, EOS và completion stopping.
4. Tách SUV mention, SUV numeric correctness và unsupported SUV; báo denominator eligible.
5. Blinded review bởi tối thiểu hai bác sĩ y học hạt nhân, adjudication người thứ ba khi bất đồng.
6. Review cùng ảnh PET/CT đầy đủ để xác lập image-grounded correctness.
7. External cohort khác thời gian/cơ sở/protocol; sau đó mới đánh giá generalizability.
8. Nếu hướng tới hỗ trợ bác sĩ: so sánh physician-only với physician+AI, editing burden và clinically significant error rate.

## 8. Trạng thái theo chuẩn khắt khe

| Tầng bằng chứng | Trạng thái |
|---|---|
| Technical execution | PASS có điều kiện; gate adapter có mâu thuẫn |
| Common-cohort comparison | PASS |
| Text-similarity evaluation | PASS, nhưng metric thấp–trung bình |
| Section-level evaluation | **NOT RELIABLE — parser failure** |
| Rule-based clinical safety | **FAIL; cần physician confirmation** |
| Blinded physician review | Chưa thực hiện |
| Image-grounded validation | Chưa thực hiện |
| External-cohort validation | Chưa thực hiện |
| Clinical utility | Chưa thực hiện |
| Clinical readiness | **NOT ESTABLISHED** |

## Evidence files

- [Bảng tổng hợp](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_3version_summary.csv)
- [Paired bootstrap](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_3version_paired_bootstrap.csv)
- [Parser diagnostics](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_parser_diagnostics.csv)
- [Notebook audit](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_3version_notebook_audit.csv)
- [Blinded physician review manifest](file:///Users/dungtuan/Downloads/ViMED-PET:CT/Mistral-7b/Stage03_3version_audit_evidence/stage03_blinded_physician_review_manifest.csv)
