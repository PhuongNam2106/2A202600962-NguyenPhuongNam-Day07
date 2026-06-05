# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** [Tên sinh viên]  
**Nhóm:** [Tên nhóm]  
**Ngày:** [Ngày nộp]

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
High cosine similarity nghĩa là hai vector embedding có hướng gần nhau. Nói cách khác, hai text chunks có thể dùng từ khác nhau nhưng vẫn đang nói về cùng chủ đề hoặc cùng ý nghĩa.

**Ví dụ HIGH similarity:**
- Sentence A: Document Chunk Embed Store Query Inject is a retrieval pipeline.
- Sentence B: The retrieval pipeline chunks documents and stores embeddings for query injection.
- Tại sao tương đồng: Cả hai câu đều nói về cùng pipeline retrieval và có chung các khái niệm document, chunk, embedding, query.

**Ví dụ LOW similarity:**
- Sentence A: Metadata filters by category and freshness.
- Sentence B: The weather in Hanoi is rainy today.
- Tại sao khác: Hai câu không chia sẻ chủ đề, mục đích, hay ý nghĩa chính.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine similarity tập trung vào hướng của vector, nên phù hợp khi cần so sánh ý nghĩa của text. Euclidean distance bị ảnh hưởng nhiều hơn bởi độ lớn vector, trong khi retrieval thường cần biết hai vector có cùng hướng semantic hay không.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**

```text
num_chunks = ceil((doc_length - overlap) / (chunk_size - overlap))
num_chunks = ceil((10000 - 50) / (500 - 50))
num_chunks = ceil(9950 / 450)
num_chunks = 23
```

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**

```text
num_chunks = ceil((10000 - 100) / (500 - 100))
num_chunks = ceil(9900 / 400)
num_chunks = 25
```

Overlap tăng làm số chunk tăng từ 23 lên 25. Overlap nhiều hơn giúp giữ ngữ cảnh giữa hai chunk liên tiếp, nhưng cũng làm tăng số lượng chunk cần embed và có thể tạo trùng lặp.

---

## 2. Document Selection - Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** AI in Action lecture slides - Day05 AI product design and Day07 data foundations

Nhóm dùng 2 slide deck thật từ folder `D:\AI Invidual Tutor\slides`: Day05 về thiết kế sản phẩm AI trong môi trường bất định, và Day07 về data foundations, embedding, vector store, chunking, retrieval. Hai slide deck này phù hợp với Phase 2 vì chúng có nội dung liên quan trực tiếp đến AI product, data strategy và retrieval-enabled agent.

Text được trích từ PDF bằng `pypdf` và lưu thành 2 file `.txt` trong `data/` để dùng cho chunking và benchmark.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | `day05_lecture_slides.txt` | `D:\AI Invidual Tutor\slides\05-day05-lecture-slides.pdf` | 27152 | `day=05`, `category=ai_product_design`, `source_type=lecture_slide`, `language=vi/en` |
| 2 | `day07_data_foundations_slides.txt` | `D:\AI Invidual Tutor\slides\07-day07-data-foundations-embedding-vector-store.pdf` | 29640 | `day=07`, `category=data_foundations`, `source_type=lecture_slide`, `language=vi/en` |

Although there are 2 source files, the retrieval index uses slide-level chunks: 56 chunks from Day05 and 70 chunks from Day07, total 126 retrieval units.

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|--------------------------------|
| `day` | string | `05`, `07` | Lọc đúng bài học khi query chỉ hỏi về Day05 hoặc Day07. |
| `category` | string | `ai_product_design`, `data_foundations` | Phân biệt nội dung product/UX/eval với nội dung embedding/vector store. |
| `source_type` | string | `lecture_slide` | Ghi rõ đây là nguồn slide, khác với docs, policy, FAQ. |
| `slide` | integer | `37` | Truy vết kết quả về slide cụ thể để verify. |
| `source` | string | `data/day07_data_foundations_slides.txt` | Cho biết chunk đến từ file text nào. |

---

## 3. Chunking Strategy - Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` với `chunk_size=700` trên 2 file text trích từ slides:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|----------|----------|-------------|------------|--------------------|
| `day05_lecture_slides.txt` | FixedSizeChunker (`fixed_size`) | 39 | 696.2 | Trung bình: chunk đều kích thước nhưng có thể cắt ngang slide. |
| `day05_lecture_slides.txt` | SentenceChunker (`by_sentences`) | 54 | 501.7 | Khá tốt, nhưng slide PDF có nhiều dòng OCR/fragment nên tách câu không ổn định. |
| `day05_lecture_slides.txt` | RecursiveChunker (`recursive`) | 57 | 474.6 | Tốt hơn fixed-size vì có thể tận dụng separator và heading slide. |
| `day07_data_foundations_slides.txt` | FixedSizeChunker (`fixed_size`) | 43 | 689.3 | Trung bình: có thể trộn nội dung giữa các slide liên tiếp. |
| `day07_data_foundations_slides.txt` | SentenceChunker (`by_sentences`) | 65 | 455.0 | Khá tốt với slide nhiều câu, nhưng vẫn bị nhiều dòng ngắn/OCR. |
| `day07_data_foundations_slides.txt` | RecursiveChunker (`recursive`) | 60 | 492.0 | Tốt: giữ được nhiều section và heading từ text PDF. |

### Strategy Của Tôi

**Loại:** Custom slide-section chunking + metadata filtering

**Mô tả cách hoạt động:**  
Sau khi trích text từ PDF, mỗi slide được đánh dấu bằng heading `## Slide N`. Strategy của tôi tách text theo các heading này, coi mỗi slide là một retrieval chunk tự nhiên. Mỗi chunk có metadata `day`, `category`, `source_type`, `slide`, và `source`, nên khi query hỏi về Day05/Day07 có thể filter đúng bài học trước khi search.

**Tại sao tôi chọn strategy này cho domain nhóm?**  
Slide deck có cấu trúc rất rõ: mỗi slide thường trình bày một ý chính. Nếu cắt theo fixed-size, một chunk có thể cắt ngang slide hoặc trộn 2 slide khác nhau. Tách theo slide giúp chunk có ý nghĩa hơn, dễ verify hơn, và rất tiện cho citation vì có thể trả về `slide=37` hoặc `slide=9`.

**Code snippet (custom strategy ý tưởng):**

```python
import re

slide_pattern = re.compile(r"^## Slide (\d+)\n", re.MULTILINE)

def split_slides(text: str) -> list[tuple[int, str]]:
    matches = list(slide_pattern.finditer(text))
    slides = []
    for i, match in enumerate(matches):
        start = match.end()
        end = matches[i + 1].start() if i + 1 < len(matches) else len(text)
        slide_no = int(match.group(1))
        content = text[start:end].strip()
        if content:
            slides.append((slide_no, content))
    return slides
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|----------|----------|-------------|------------|--------------------|
| `day05_lecture_slides.txt` | best baseline: `recursive` | 57 | 474.6 | Tốt, nhưng không đảm bảo mỗi chunk trùng với một slide. |
| `day05_lecture_slides.txt` | của tôi: slide-section chunks | 56 | about 485 | Tốt hơn cho slide deck vì mỗi chunk có `slide` metadata. |
| `day07_data_foundations_slides.txt` | best baseline: `recursive` | 60 | 492.0 | Tốt, nhưng một số chunk có thể gộp/tách khác slide boundary. |
| `day07_data_foundations_slides.txt` | của tôi: slide-section chunks | 70 | about 423 | Tốt hơn cho benchmark vì top result có slide number rõ ràng. |

### So Sánh Với Thành Viên Khác

Chưa có kết quả thật từ các thành viên khác, nên bảng này ghi candidate strategies để nhóm có thể cập nhật khi mỗi người chạy cùng 5 benchmark queries.

| Thành viên / Candidate | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|------------------------|----------|----------------------|-----------|----------|
| Tôi | Slide-section chunks + metadata filter | 10/10 | Chunk trùng với đơn vị slide, dễ verify, có `day` và `slide`. | Nếu một slide quá dài/nhiều ý thì cần split tiếp. |
| Candidate A | FixedSizeChunker | Chưa chạy | Đơn giản, chunk đều kích thước. | Dễ cắt ngang slide và mất ngữ cảnh. |
| Candidate B | RecursiveChunker | Chưa chạy | Khai thác separator/heading, ổn với Markdown/text. | Không đảm bảo chunk trùng với slide number. |

**Strategy nào tốt nhất cho domain này? Tại sao?**  
Với data là lecture slides, slide-section chunking tốt nhất vì boundary của slide là boundary ý nghĩa tự nhiên. Metadata `day` và `slide` làm cho retrieval có tính traceable: khi kết quả trả về, có thể chỉ ra ngay slide nào ủng hộ câu trả lời.

---

## 4. My Approach - Cá nhân (10 điểm)

Giải thích cách tiếp cận khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk` - approach:**  
Hàm dùng regex `(?<=[.!?])\s+` để tách text tại khoảng trắng sau dấu kết câu. Sau đó gom mỗi `max_sentences_per_chunk` câu thành một chunk và strip whitespace để tránh chunk rỗng.

**`RecursiveChunker.chunk` / `_split` - approach:**  
Hàm `chunk` xử lý base case trước: text rỗng thì trả `[]`, text ngắn hơn `chunk_size` thì trả một chunk. Nếu text quá dài, `_split` thử separator theo thứ tự `\n\n`, `\n`, `. `, space, và fallback fixed-size khi không tách được nữa.

### EmbeddingStore

**`add_documents` + `search` - approach:**  
Mỗi document được lưu thành record gồm `id`, `content`, `metadata`, và `embedding`. Khi search, query được embed, tính dot product với từng record embedding, sort giảm dần theo score, và trả về top-k.

**`search_with_filter` + `delete_document` - approach:**  
`search_with_filter` lọc metadata trước rồi mới search trong candidate records. `delete_document` xóa các record có `metadata["doc_id"]` trùng với id cần xóa và trả `True` nếu có record bị xóa.

### KnowledgeBaseAgent

**`answer` - approach:**  
Agent gọi `store.search(question, top_k)`, ghép các chunk retrieve được thành context, tạo prompt gồm context và question, rồi gọi `llm_fn(prompt)`. Cách này là RAG pattern tối thiểu: retrieve trước, generate sau.

### Test Results

```text
python -m pytest tests/ -v
42 passed
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions - Cá nhân (5 điểm)

Actual score được tính bằng `compute_similarity()` trên lexical embedding offline dùng riêng cho benchmark slides.

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|------------|------------|---------|--------------|-------|
| 1 | A working agent is not automatically a successful AI product. | An AI product needs user trust beyond a working agent. | high | 0.707 | Yes |
| 2 | Document Chunk Embed Store Query Inject is a retrieval pipeline. | The retrieval pipeline chunks documents and stores embeddings for query injection. | high | 0.354 | Yes |
| 3 | Knowledge data is useful for retrieval. | Operational data often needs controlled API or SQL access. | medium | 0.224 | Yes |
| 4 | AI product design needs evaluation under uncertainty. | The weather in Hanoi is rainy today. | low | 0.000 | Yes |
| 5 | Metadata filters by category and freshness. | A prompt can be longer than the context window. | low | 0.000 | Yes |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Pair 3 có score medium vì hai câu cùng nằm trong topic "data agent cần", nhưng nói về hai loại data khác nhau. Điều này cho thấy retrieval không chỉ cần tìm document cùng chủ đề, mà còn cần chunk đủ rõ để phân biệt đúng nội dung cần trả lời.

---

## 6. Results - Cá nhân (10 điểm)

Benchmark dùng 2 slide deck đã trích text, custom slide-section chunking, metadata filter theo `day`, và lexical embedding offline. `_mock_embed` trong lab tốt cho unit test nhưng không hiểu nghĩa, nên benchmark dùng lexical embedding để kết quả dễ giải thích hơn.

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Why does a working agent not automatically become a successful product? | A working agent is not enough: the product must prove user need, trust, UX, requirements and evaluation under uncertainty. |
| 2 | Requirement UX Eval 3 pillars | The three pillars are Requirement, UX, and Eval. |
| 3 | What is the Document Chunk Embed Store Query Inject pipeline? | Document -> Chunk -> Embed -> Store -> Query -> Inject for retrieval-enabled answering. |
| 4 | What three types of data does an agent need? | Knowledge data, operational data, and contextual data. |
| 5 | Why do chunk quality and metadata matter for retrieval quality? | Bad OCR, stale policy, chunks cut mid-sentence and missing metadata cause hallucination; clean text, source, section chunks and filters improve grounded answers. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|---------------------------------|-------|-----------|-------------------------|
| 1 | Working agent vs successful product | Day05 slide 5: "Agent của bạn chạy được rồi. Vậy tại sao có thể không ai dùng?" | 0.167 | Yes | Agent running is only a technical milestone; product success needs user need and trust. |
| 2 | Requirement UX Eval 3 pillars | Day05 slide 17: "3 trụ: Requirement - UX - Eval" | 0.500 | Yes | The three pillars are Requirement, UX, and Eval. |
| 3 | Document Chunk Embed Store Query Inject | Day07 slide 37: retrieval pipeline Document -> Chunk -> Embed -> Store -> Query -> Inject | 0.350 | Yes | Retrieval pipeline chunks docs, embeds them, stores vectors, queries, then injects context. |
| 4 | Three types of agent data | Day07 slide 9: Knowledge Data, Operational Data, Contextual Data | 0.537 | Yes | Agents need knowledge, operational, and contextual data. |
| 5 | Chunk quality and metadata | Day07 slide 13: Data Quality Pyramid, raw/cleaned/structured/enriched | 0.292 | Yes | Clean data, chunk structure and metadata improve retrieval; bad OCR/no metadata causes hallucination. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

### Top-3 Evidence Summary

| # | Top-3 Evidence |
|---|----------------|
| 1 | Day05 slides 5, 6, 3: agent chạy được, product thành công, mục tiêu Day05. |
| 2 | Day05 slides 17, 29, 19: Requirement, UX, Eval and AI spec. |
| 3 | Day07 slides 37, 4, 51: retrieval pipeline and lab embed/store step. |
| 4 | Day07 slides 9, 36, 3: 3 loại data và kết nối agent với data. |
| 5 | Day07 slides 13, 51, 37: data quality pyramid, embed/store, retrieval pipeline. |

---

## 7. What I Learned (5 điểm - Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Chưa có số liệu thật từ thành viên khác. Nếu nhóm chạy tiếp, tôi muốn so sánh slide-section chunking với recursive chunking và fixed-size chunking trên cùng 5 benchmark queries.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  
Chưa có demo nhóm khác. Điểm tôi sẽ quan sát là cách các nhóm xử lý source tài liệu có cấu trúc đặc biệt như slide, PDF OCR, hoặc table.

**Failure case và đề xuất cải thiện:**  
Failure/partial case là Q1: score top-1 chỉ 0.167, thấp hơn các query khác. Lý do là query bằng tiếng Anh trong khi slide Day05 chủ yếu tiếng Việt, nên lexical embedding chỉ bắt được một phần từ khóa như "agent/product". Cách cải thiện là dùng embedding model đa ngôn ngữ thật, hoặc viết query benchmark cùng ngôn ngữ với source slide.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Tôi sẽ clean footer/OCR noise trong PDF extraction, chuẩn hóa tiếng Việt, và merge các dòng bị tách ký tự. Tôi cũng sẽ thêm metadata `section_title` để filter theo phần bài học như `data_strategy`, `embedding`, `vector_store`, `eval`, hoặc `ux`.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 8 / 10 |
| Chunking strategy | Nhóm | 14 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 9 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 3 / 5 |
| **Tổng** | | **84 / 100** |
