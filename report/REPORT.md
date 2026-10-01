# Báo cáo triển khai — Hệ thống đánh giá chất lượng Trợ lý ảo AI

| Thông tin            | Nội dung                                                                                                      |
| -------------------- | ------------------------------------------------------------------------------------------------------------- |
| **Dự án**            | OrbitTech Store — Trợ lý ảo hỗ trợ khách hàng (RAG Chatbot)                                                   |
| **Hạng mục**         | Ngày 14 — AI Evaluation & Benchmarking Pipeline                                                               |
| **Phạm vi báo cáo**  | CP0 → CP5 (Task 1 đến Task 6)                                                                                 |
| **Người thực hiện**  | Phan Duy Thanh — 2A202602930                                                                                  |
| **Kết quả kiểm thử** | ✅ 42/42 bài kiểm thử đạt (0 lỗi)                                                                              |
| **Trạng thái**       | ✅ Hoàn thành toàn bộ CP0–CP5 — benchmark thật đã chạy, `exercises.md` và `reflection.md` đã điền số liệu thật |

---

## 1. Tóm tắt điều hành (Executive Summary)

Chúng tôi đã xây dựng xong **bộ máy chấm điểm tự động** cho trợ lý ảo của OrbitTech Store. Nói một cách đơn giản: thay vì để nhân viên đọc thủ công hàng trăm câu trả lời của chatbot rồi tự đoán xem câu nào tốt, câu nào kém, hệ thống này **tự động cho điểm** từng câu trả lời theo những tiêu chí đã thống nhất trước, **tự động phát hiện câu trả lời kém**, **tự động tìm nguyên nhân gốc** và **đề xuất hướng khắc phục**.

Bộ máy này hoạt động như một **"phòng kiểm định chất lượng" (QC)** cho AI: chạy nhanh, lặp lại được, cho kết quả nhất quán, và — quan trọng nhất — **có thể gắn vào quy trình phát hành sản phẩm để tự động chặn các bản cập nhật làm giảm chất lượng**.

> **Điểm cốt lõi dành cho người đọc không chuyên:** Chúng tôi không cải thiện chatbot. Chúng tôi xây dựng **thước đo** để biết chatbot đang tốt lên hay xấu đi. Muốn cải tiến được, trước tiên phải đo được.

---

## 2. Vấn đề nghiệp vụ (Business Context)

| Câu hỏi nghiệp vụ                                                       | Vì sao quan trọng                                                                            |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Làm sao biết câu trả lời của chatbot là **đúng** hay **bịa** thông tin? | Trả lời sai về bảo hành, hoàn tiền, vận chuyển có thể gây thiệt hại tài chính và mất uy tín. |
| Làm sao biết chatbot có **trả lời đúng câu khách hỏi**?                 | Một câu trả lời dài dòng nhưng lạc đề không có giá trị với khách hàng.                       |
| Làm sao biết câu trả lời đã **đầy đủ** chưa?                            | Thiếu một điều kiện/ngoại lệ có thể khiến khách hiểu sai chính sách.                         |
| Làm sao đảm bảo chất lượng **không tụt** sau mỗi lần cập nhật?          | Không có "chốt chặn", một thay đổi nhỏ có thể âm thầm làm giảm chất lượng toàn hệ thống.     |

Nếu không có thước đo tự động, mọi đánh giá đều **chủ quan, tốn thời gian và không thể lặp lại** — nghĩa là không thể cải tiến một cách có kỷ luật.

---

## 3. Giải pháp đã xây dựng (theo từng Checkpoint)

### CP1 — Bộ khung dữ liệu (Task 1): "Chuẩn hóa cách ghi và cách chấm điểm"

**Đã làm:** Định hình hai "biểu mẫu" chuẩn cho toàn bộ hoạt động đánh giá:

- **Phiếu câu hỏi mẫu (Golden Question)** — mỗi câu hỏi kiểm thử gồm: câu hỏi, câu trả lời chuẩn do chuyên gia viết, tài liệu nguồn, thông tin phân loại (độ khó/nhóm chủ đề), và danh sách các đoạn tài liệu hệ thống đã tìm về.
- **Phiếu kết quả chấm điểm (Scorecard)** — mỗi lần chấm một câu trả lời sẽ ghi lại: câu trả lời thực tế của chatbot, điểm 3 tiêu chí chất lượng câu trả lời, kết luận Đạt/Không đạt, loại lỗi (nếu có), và điểm 2 tiêu chí chất lượng khâu tìm tài liệu.

**Điểm nghiệp vụ quan trọng:** Chúng tôi tách bạch rõ **hai nhóm điểm**:
- **Điểm chất lượng câu trả lời** (3 tiêu chí) → dùng để kết luận Đạt/Không đạt.
- **Điểm chất lượng khâu tìm tài liệu** (2 tiêu chí) → chỉ dùng để **chẩn đoán nguyên nhân**, giúp trả lời câu hỏi "lỗi tại chatbot hay lỗi tại khâu lấy tài liệu?".

Ngoài ra, nguyên tắc tính **Điểm tổng hợp** được chuẩn hóa: trung bình cộng của 3 tiêu chí chất lượng câu trả lời.

📌 *Giá trị:* Từ nay mọi người trong dự án nói cùng một "ngôn ngữ", số liệu so sánh được giữa các lần chạy.

---

### CP2 — Thước đo chất lượng & Trọng tài AI (Tasks 2–3)

#### 2a. Năm thước đo chất lượng (Task 2)

Chúng tôi xây dựng **5 thước đo**, chia làm hai tầng — giống như kiểm tra một dây chuyền sản xuất từ khâu đầu đến khâu cuối:

| Thước đo                                      | Câu hỏi nghiệp vụ nó trả lời                      | Cách hiểu đơn giản                                                       |
| --------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------ |
| **Faithfulness** (Độ trung thực)              | Chatbot có bịa thông tin không?                   | Thợ có dùng đúng nguyên liệu được cấp, hay tự chế thêm?                  |
| **Relevance** (Độ liên quan)                  | Câu trả lời có đúng trọng tâm câu hỏi?            | Khách hỏi A, nhân viên trả lời đúng A hay lan sang B?                    |
| **Completeness** (Độ đầy đủ)                  | Câu trả lời có thiếu ý quan trọng?                | Trả lời có đủ điều kiện, ngoại lệ, mốc thời gian chưa?                   |
| **Context Recall** (Độ phủ tài liệu)          | Máy có **tìm đủ** tài liệu liên quan không?       | Thủ kho có lấy đủ hàng cần thiết không?                                  |
| **Context Precision** (Độ chính xác thứ hạng) | Tài liệu **liên quan có được xếp lên đầu** không? | Hàng cần dùng có bày sẵn trên kệ trước, hay bị chôn dưới đống hàng khác? |

Đặc biệt, **điểm chất lượng tìm tài liệu có tính đến thứ hạng**: bộ máy thưởng điểm cao khi các đoạn tài liệu liên quan được xếp ở vị trí đầu — phản ánh đúng trải nghiệm thực tế (tài liệu đúng mà bị chôn sâu thì cũng gần như vô dụng).

**Cơ chế tự động phân loại lỗi:** Hệ thống tự gán nhãn loại lỗi dựa trên tiêu chí yếu nhất — ví dụ bịa thông tin → `hallucination`; lạc đề → `irrelevant`; thiếu ý → `incomplete`. Việc này giúp khâu phân tích phía sau **phân nhóm lỗi tự động** thay vì đọc thủ công.

#### 2b. Trọng tài AI có kiểm soát thiên kiến (Task 3)

Ngoài các thước đo bằng từ khóa, chúng tôi bổ sung một **"giám khảo AI"** dùng mô hình ngôn ngữ để chấm câu trả lời theo **bảng tiêu chí (rubric)**.

Vì giám khảo là AI, nó có thể mắc thiên kiến giống con người, nên chúng tôi trang bị **bộ kiểm tra 3 loại thiên kiến**:

| Loại thiên kiến                        | Rủi ro nghiệp vụ                                         | Cách hệ thống phát hiện                     |
| -------------------------------------- | -------------------------------------------------------- | ------------------------------------------- |
| **Positional bias** (ưu ái vị trí đầu) | Luôn chấm cao phương án xuất hiện trước, bất kể nội dung | So sánh điểm giữa các phương án theo thứ tự |
| **Leniency bias** (quá dễ dãi)         | Cho điểm cao tràn lan, làm mất tác dụng cảnh báo         | Cảnh báo khi điểm trung bình > 0.8          |
| **Severity bias** (quá khắt khe)       | Chê bai mọi câu trả lời, gây báo động giả                | Cảnh báo khi điểm trung bình < 0.3          |

📌 *Giá trị:* Bộ máy đánh giá **tự biết khi nào nó không đáng tin** — một yêu cầu bắt buộc để dùng AI làm giám khảo một cách có trách nhiệm.

---

### CP3 — Điều phối benchmark & Phân tích lỗi (Tasks 4–5)

#### 4. Bộ điều phối đo lường (Task 4)

Đây là "nhạc trưởng" tự động hóa toàn bộ quy trình:

1. **Chạy hàng loạt:** Đưa từng câu hỏi mẫu qua chatbot, thu câu trả lời, chấm điểm ngay.
2. **Báo cáo tổng hợp:** Tính tỷ lệ đạt, điểm trung bình từng tiêu chí, và bảng thống kê các loại lỗi.
3. **Phát hiện giảm chất lượng (Regression):** So sánh với lần chạy trước; **cảnh báo khi bất kỳ tiêu chí nào giảm quá 0.05**. Đây chính là cơ chế **"chốt chất lượng" trong CI/CD** — tự động chặn phát hành bản cập nhật làm hỏng chất lượng.
4. **Lọc danh sách lỗi:** Trích ra đúng những câu trả lời dưới ngưỡng để chuyển sang phân tích.

#### 5. Bộ phân tích nguyên nhân lỗi (Task 5)

Khi đã có danh sách câu trả lời kém, bộ phận này **biến dữ liệu thô thành quyết định hành động**:

- **Phân nhóm lỗi:** Đếm và gom lỗi theo loại → nhìn ra **lỗi nào phổ biến nhất** thay vì xử lý lẻ tẻ.
- **Chẩn đoán nguyên nhân gốc:** Dựa trên tiêu chí yếu nhất, chỉ ra nguyên nhân khả dĩ
  - *Điểm trung thực thấp* → "Tài liệu nguồn thiếu/không liên quan — cải thiện khâu tìm kiếm"
  - *Độ liên quan thấp* → "Câu trả lời không đúng trọng tâm — làm rõ hướng dẫn cho AI"
  - *Độ đầy đủ thấp* → "Thiếu thông tin quan trọng — tăng lượng tài liệu cấp cho AI"
  - *Nhiều tiêu chí cùng yếu* → "Lỗi đa điểm — rà soát toàn bộ quy trình"
- **Đề xuất cải tiến:** Sinh tối thiểu 3 hành động cụ thể, ưu tiên theo mức độ phổ biến của lỗi.
- **Sổ nhật ký cải tiến (Improvement Log):** Xuất bảng Markdown ghi rõ *mã lỗi – loại – nguyên nhân – cách khắc phục – trạng thái*, dùng được ngay làm tài liệu theo dõi công việc.

📌 *Giá trị:* Vòng lặp cải tiến liên tục trở nên **khép kín và tự động**: Đo → Phân tích → Khắc phục → Bổ sung dữ liệu → Lặp lại.

---

### CP0 — Chuẩn bị môi trường: "Đảm bảo sân chơi sẵn sàng trước khi đo"

**Đã làm:** Dựng môi trường chạy (Python 3.11+, cài đủ thư viện), tạo file cấu hình `.env` từ `.env.example`, và chạy bộ kiểm thử nền tảng để xác nhận hệ thống "chạy được" trước khi bắt tay vào code.

**Vì sao quan trọng (góc nhìn BA):** Giống như trước khi kiểm kê hàng hoá phải đảm bảo cân và sổ sách hoạt động. Nếu môi trường lỗi, mọi con số về sau đều không đáng tin. Bước này xác lập **"đường cơ sở" (baseline)** để so sánh.

📌 *Giá trị:* Mọi kết quả sau này đều có điểm so sánh chuẩn — biết được là đang tốt lên hay xấu đi.

---

### CP4 — Bộ đề kiểm thử chuẩn (Task 6): "Xây đề thi có đáp án"

Đây là bước tạo **thước đo đối chiếu**: một bộ 20 câu hỏi mẫu kèm đáp án chuẩn do chuyên gia viết, trải đều theo 4 mức độ khó và phủ toàn bộ 10 tài liệu nghiệp vụ.

| Hạng mục                                                    | Yêu cầu | Kết quả thực tế |
| ----------------------------------------------------------- | ------- | --------------- |
| Tổng số câu hỏi                                             | 20      | 20 / 20         |
| Câu dễ (tra cứu trực tiếp)                                  | 5       | 5 / 5           |
| Câu trung bình (ghép nhiều nguồn)                           | 7       | 7 / 7           |
| Câu khó (nhiều điều kiện/ngoại lệ)                          | 5       | 5 / 5           |
| Câu đối kháng (out-of-scope, prompt injection, tiền đề sai) | 3       | 3 / 3           |
| Tài liệu nghiệp vụ được phủ                                 | 10      | 10 / 10         |
| Kiểm tra tự động (validator)                                | PASS    | ✅ PASS          |

**Điểm nghiệp vụ quan trọng:** Mỗi đáp án chuẩn đều phải có **bằng chứng nguyên văn** trích từ tài liệu nguồn — máy kiểm tra tự động xác nhận điều này. Nói cách khác, không có "đáp án theo cảm tính": mọi khẳng định đều truy được về tài liệu gốc.

**Kết quả kiểm tra tự động thực tế:**

```text
QA pairs: 20
Difficulty: easy=5, medium=7, hard=5, adversarial=3
Document coverage: 10/10
PASS: dataset structure and evidence provenance are valid.
```

**Chạy trợ lý ảo thật và benchmark:** ✅ **ĐÃ HOÀN THÀNH.** Vì chỉ có khoá Gemini, `domain_assistant.py` được bổ sung một nhánh **OpenAI-compatible** (tự nhận diện model `gemini-*`) và **giữ nguyên toàn bộ logic BM25 retrieval + prompt**; `.env` được đổi model sang `gemini-3.5-flash-lite`.

```text
Generated 20 actual answers -> artifacts/actual_answers.json
Overall pass rate: 50.0% (10/20)
Avg Context Recall: 0.948 | Avg Context Precision: 0.950
Avg Faithfulness: 0.751 | Avg Relevance: 0.578 | Avg Completeness: 0.654
Failure types: off_topic 7, incomplete 1, irrelevant 1, hallucination 1
3 lowest: A02 (0.065), A01 (0.415), E01 (0.496)
```

**Đọc kết quả theo góc nhìn nghiệp vụ:** khâu tra cứu tài liệu gần như hoàn hảo (≈0.95) → nút thắt nằm ở **khâu sinh câu trả lời** (trả lời quá ngắn/thiếu chi tiết). Đặc biệt, 2 ca điểm thấp nhất (A01, A02) lại là các **hành vi an toàn đúng đắn** — chatbot từ chối yêu cầu tiết lộ dữ liệu và từ chối câu hỏi ngoài phạm vi — nhưng bị thước đo trùng từ khoá chấm sai. Đây là phát hiện quan trọng, đã ghi rõ trong `reflection.md`.

---

### CP5 — Phân tích lỗi & Đóng gói bài nộp: "Biến số liệu thành hành động"

Gồm 2 việc: (1) phân tích 3 ca lỗi điển hình bằng phương pháp 5 Whys và viết chiến lược chống giảm chất lượng trong `reflection.md`; (2) đóng gói bài nộp (đồng bộ `solution/solution.py`, kiểm tra test, kiểm tra không lộ thông tin bí mật).

**Trạng thái:** ✅ **Hoàn thành.** `reflection.md` đã có 3 phân tích 5 Whys dựa trên trace thật, bảng phân loại lỗi, improvement log, chiến lược regression và vòng cải tiến liên tục. Bài nộp đã đóng gói: test 42/42, `solution/solution.py` đồng bộ với `template.py`, `.env` không bị git theo dõi.

📌 *Giá trị:* Đây là mắt xích biến "báo cáo số" thành "kế hoạch hành động" — đội sản phẩm biết chính xác cần sửa gì tiếp theo.

---

## 4. Giá trị nghiệp vụ tổng hợp

| Năng lực mới                      | Trước đây              | Sau khi triển khai                           |
| --------------------------------- | ---------------------- | -------------------------------------------- |
| Đánh giá câu trả lời              | Đọc thủ công, cảm tính | Tự động, nhất quán, chạy trong vài giây      |
| Phát hiện bịa thông tin           | Chỉ khi khách phàn nàn | Phát hiện trước khi phát hành                |
| So sánh giữa các phiên bản        | Không có cơ sở         | Định lượng, cảnh báo tự động khi giảm > 0.05 |
| Tìm nguyên nhân gốc               | Suy đoán, tranh luận   | Chẩn đoán dựa trên dữ liệu                   |
| Kiểm soát chất lượng giám khảo AI | Không kiểm tra         | Cảnh báo 3 loại thiên kiến                   |
| Chốt chặn phát hành (CI/CD)       | Không có               | Cổng chất lượng tự động                      |

---

## 5. Tiêu chí nghiệm thu & Kết quả kiểm thử

Mỗi hạng mục đều có **tiêu chí nghiệm thu (Acceptance Criteria)** và được kiểm chứng bằng bộ kiểm thử tự động.

| #    | Hạng mục                                       | Tiêu chí nghiệm thu                                                 | Kết quả     |
| ---- | ---------------------------------------------- | ------------------------------------------------------------------- | ----------- |
| CP1  | Bộ khung dữ liệu + Điểm tổng hợp               | Điểm tổng hợp = trung bình 3 tiêu chí                               | ✅ 3/3       |
| CP2a | 5 thước đo chất lượng                          | Mỗi thước đo trả về giá trị hợp lệ [0–1]; thứ hạng được thưởng điểm | ✅ 14/14     |
| CP2b | Kết nối thước đo tìm tài liệu                  | Không có tài liệu → để trống; có tài liệu → tính & lưu              | ✅ 3/3       |
| CP2c | Trọng tài AI + kiểm soát thiên kiến            | Chấm điểm theo rubric; trả về 3 cờ thiên kiến                       | ✅ 4/4       |
| CP3a | Bộ điều phối + báo cáo + chống giảm chất lượng | Đủ chỉ số báo cáo; phát hiện giảm > 0.05                            | ✅ 11/11     |
| CP3b | Phân tích lỗi + nhật ký cải tiến               | Phân nhóm lỗi; ≥ 3 đề xuất; bảng Markdown hợp lệ                    | ✅ 9/9       |
| —    | Bonus — Xếp hạng lại tài liệu (Exercise 3.5)   | Cải thiện/giữ nguyên độ chính xác thứ hạng                          | ✅ 1/1       |
|      | **TỔNG**                                       | **Toàn bộ bài kiểm thử đạt**                                        | **✅ 42/42** |

> **Ghi chú kỹ thuật:** Do phần bonus xếp hạng lại tài liệu đã được triển khai, kết quả kiểm thử đạt **42/42** (thay vì 41 đạt + 1 bỏ qua như mức tối thiểu bắt buộc). Đây là điểm cộng (Exercise 3.5), không ảnh hưởng đến phần bắt buộc.

### Kiểm chứng bổ sung cho các checkpoint còn lại

| Checkpoint | Sản phẩm                                             | Bằng chứng kiểm chứng                                                       | Trạng thái   |
| ---------- | ---------------------------------------------------- | --------------------------------------------------------------------------- | ------------ |
| **CP0**    | Môi trường + `.env` + baseline test                  | Bộ kiểm thử chạy được, 42 bài được thu thập                                 | ✅ Hoàn thành |
| **CP4**    | Bộ đề 20 câu + đáp án chuẩn có bằng chứng nguyên văn | Validator PASS (20/20, 10/10); benchmark chạy thật: pass rate 50%, 5 metric | ✅ Hoàn thành |
| **CP5**    | `reflection.md` + đóng gói bài nộp                   | 3 phân tích 5 Whys, improvement log, regression strategy; test 42/42        | ✅ Hoàn thành |

---

## 6. Từ điển thuật ngữ (dành cho người đọc không chuyên)

| Thuật ngữ                        | Nghĩa đời thường                                                                                                                          |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **RAG**                          | Cách trợ lý ảo trả lời: tra cứu tài liệu liên quan trước, rồi dựa vào đó để trả lời — giống nhân viên tra sổ tay trước khi trả lời khách. |
| **Golden Dataset**               | Bộ đề kiểm tra chuẩn, có đáp án mẫu do chuyên gia viết — như "đề thi có đáp án" để chấm điểm.                                             |
| **Metric (Thước đo)**            | Tiêu chí cho điểm, ví dụ "độ trung thực", "độ đầy đủ".                                                                                    |
| **LLM-as-a-Judge**               | Dùng một AI khác làm "giám khảo" để chấm điểm câu trả lời.                                                                                |
| **Rubric**                       | Bảng tiêu chí chấm điểm, quy định rõ thế nào là điểm cao/thấp.                                                                            |
| **Bias (Thiên kiến)**            | Xu hướng chấm điểm lệch lạc, ví dụ luôn ưu ái câu trả lời dài.                                                                            |
| **Regression (Giảm chất lượng)** | Chất lượng tụt so với phiên bản trước đó.                                                                                                 |
| **CI/CD Quality Gate**           | Cổng kiểm soát: nếu chất lượng không đạt ngưỡng thì không cho phát hành.                                                                  |
| **5 Whys**                       | Kỹ thuật hỏi "Tại sao?" liên tiếp để truy tìm nguyên nhân gốc.                                                                            |

---

## 7. Rủi ro, giới hạn & Khuyến nghị

| Hạng mục                  | Nội dung                                                                                         | Khuyến nghị                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| **Phương pháp chấm điểm** | Các thước đo hiện dùng **trùng khớp từ khóa** — nhanh, minh bạch, nhưng chưa hiểu ngữ nghĩa sâu. | Ở môi trường thật, thay/thêm mô hình ngôn ngữ hoặc framework chuyên dụng (RAGAS, DeepEval). |
| **Giám khảo AI**          | Cần hiệu chỉnh với đánh giá của con người để đảm bảo công bằng.                                  | Định kỳ so khớp điểm AI với chuyên gia (calibration).                                       |
| **Ngưỡng cảnh báo**       | Ngưỡng 0.5 (đạt/không đạt) và 0.05 (giảm chất lượng) là cấu hình mặc định.                       | Rà soát và điều chỉnh theo mức độ chịu rủi ro thực tế của từng nghiệp vụ.                   |
| **Dữ liệu kiểm thử**      | Kết quả chỉ đáng tin khi bộ đề kiểm tra đủ đa dạng và sát thực tế.                               | Hoàn thành Golden Dataset 20 câu (CP4) với đủ 10 tài liệu nguồn.                            |

---

## 8. Trạng thái & Bước tiếp theo

**Trạng thái hiện tại:** ✅ **Toàn bộ CP0–CP5 đã hoàn thành và kiểm chứng.**

| Checkpoint                  | Kết quả                                                                       |
| --------------------------- | ----------------------------------------------------------------------------- |
| CP0 — Setup                 | Môi trường chạy được, 42 test thu thập                                        |
| CP1–CP3 — Core              | 42/42 test đạt; `solution/solution.py` đồng bộ                                |
| CP4 — Dataset & Benchmark   | Validator PASS (20/20, 10/10); benchmark thật: pass rate 50%, 5 metric đầy đủ |
| CP5 — Reflection & Đóng gói | `reflection.md` hoàn chỉnh; `.env` không bị git theo dõi                      |

**Ghi chú về provider:** Do chỉ có khoá Gemini, `domain_assistant.py` được bổ sung nhánh OpenAI-compatible (tự nhận diện model `gemini-*`). Mặc định cũ (OpenAI Responses API) **không đổi** — nếu dùng lại khoá OpenAI với model `gpt-*`, hệ thống tự quay về đường cũ. Toàn bộ logic retrieval (BM25) và prompt được giữ nguyên, nên kết quả benchmark phản ánh đúng hệ thống dưới đánh giá.

---

*Báo cáo được lập theo phương pháp Business Analysis: tập trung vào giá trị nghiệp vụ, tiêu chí nghiệm thu và bằng chứng kiểm chứng — đảm bảo người đọc không chuyên vẫn nắm được toàn bộ nội dung.*
