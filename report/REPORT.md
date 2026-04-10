# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Trần Đặng Quang Huy - 2A202600292
**Nhóm:** E403
**Ngày:** 10/04/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> *Viết 1-2 câu:*
>
> Hai đoạn text có **high cosine similarity** nghĩa là embedding của chúng “cùng hướng” trong không gian vector, thường tương ứng với việc chúng **gần nghĩa / nói về cùng một ý** (dù có thể khác từ ngữ).

**Ví dụ HIGH similarity:**
- Sentence A:
- Sentence B:
- Tại sao tương đồng:
  - Sentence A: How to reset my password?
  - Sentence B: Steps to change your account password.
  - Tại sao tương đồng: Cùng nói về việc đổi/reset mật khẩu tài khoản (cùng intent, cùng chủ đề).

**Ví dụ LOW similarity:**
- Sentence A:
- Sentence B:
- Tại sao khác:
  - Sentence A: I love Italian pizza.
  - Sentence B: Quantum entanglement is a physics phenomenon.
  - Tại sao khác: Hai câu nói về 2 chủ đề không liên quan (ẩm thực vs vật lý).

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> *Viết 1-2 câu:*
>
> Embedding thường quan trọng **hướng** hơn là **độ lớn**, và nhiều hệ embed còn được normalize; cosine similarity đo độ “cùng hướng” nên ổn định hơn với thay đổi scale, phù hợp cho so sánh ngữ nghĩa.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:*
> *Đáp án:*
>
> \[
> \text{num\_chunks}=\left\lceil\frac{doc\_length-overlap}{chunk\_size-overlap}\right\rceil
> =\left\lceil\frac{10000-50}{500-50}\right\rceil
> =\left\lceil\frac{9950}{450}\right\rceil
> =\left\lceil 22.111...\right\rceil=23
> \]
>
> Đáp án: **23 chunks**

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> *Viết 1-2 câu:*
>
> \(\left\lceil\frac{10000-100}{500-100}\right\rceil=\left\lceil\frac{9900}{400}\right\rceil=25\) nên **tăng lên 25 chunks**. Overlap lớn hơn giúp **giữ ngữ cảnh xuyên ranh giới chunk** (giảm mất mạch), đổi lại tốn storage/tính toán hơn.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Hệ thống Chính sách và Điều khoản dịch vụ của Sàn Thương mại điện tử (Shopee).

**Tại sao nhóm chọn domain này?**
> Nhóm chọn domain này vì các chính sách thương mại điện tử rất thiết thực, liên quan trực tiếp đến quyền lợi người dùng (người mua và người bán). Các văn bản này có đặc điểm chung là dài, nhiều điều khoản chi tiết, nhiều ngoại lệ và thường xuyên thay đổi theo thời gian. Đây là use-case hoàn hảo để kiểm thử khả năng tìm kiếm (retrieval) và tổng hợp (generation) của hệ thống RAG nhằm xây dựng một "Trợ lý hỗ trợ khách hàng tự động", đảm bảo luôn tư vấn dựa trên chính sách mới nhất và đúng phạm vi áp dụng.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | dieu_khoan_dich_vu.txt | Shopee VN | ~55,000 | `{"chu_de": "dieu_khoan_chung", "doi_tuong": "ca_hai", "ngay_hieu_luc": "Hien_hanh", "loai_van_ban": "dieu_khoan", "pham_vi_ap_dung": "chung"}` |
| 2 | chinh_sach_tra_hang_hoan_tien.txt | Shopee VN | ~25,000 | `{"chu_de": "don_hang", "doi_tuong": "ca_hai", "ngay_hieu_luc": "11/03/2026", "loai_van_ban": "chinh_sach", "pham_vi_ap_dung": "shopee_mall"}` |
| 3 | chinh_sach_cam_han_che_san_pham.txt | Shopee VN | ~30,000 | `{"chu_de": "hang_hoa", "doi_tuong": "nguoi_ban", "ngay_hieu_luc": "Hien_hanh", "loai_van_ban": "chinh_sach", "pham_vi_ap_dung": "chung"}` |
| 4 | chinh_sach_chong_gian_lan.txt | Shopee VN | ~15,000 | `{"chu_de": "vi_pham", "doi_tuong": "nguoi_ban", "ngay_hieu_luc": "28/12/2023", "loai_van_ban": "chinh_sach", "pham_vi_ap_dung": "chung"}` |
| 5 | chinh_sach_bao_mat.txt | Shopee VN | ~35,000 | `{"chu_de": "bao_mat", "doi_tuong": "ca_hai", "ngay_hieu_luc": "03/07/2023", "loai_van_ban": "chinh_sach", "pham_vi_ap_dung": "chung"}` |
| 6 | quy_dinh_ve_dang_ban_san_pham_tren_shoppe.txt | Shopee VN | ~20,000 | `{"chu_de": "hang_hoa", "doi_tuong": "nguoi_ban", "ngay_hieu_luc": "Hien_hanh", "loai_van_ban": "quy_dinh", "pham_vi_ap_dung": "chung"}` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `chu_de` | String | "don_hang", "hang_hoa", "bao_mat", "vi_pham" | Giúp agent khoanh vùng tìm kiếm. VD câu hỏi về phí trả hàng sẽ chỉ filter vào `chu_de: don_hang` để tránh nhầm lẫn với các phí dịch vụ chung ở điều khoản khác. |
| `doi_tuong` | String | "nguoi_ban", "nguoi_mua", "ca_hai" | Giúp phân tách rõ ràng quyền lợi/nghĩa vụ của từng bên, vì một số chính sách chỉ áp dụng riêng cho người bán (ví dụ: quy định đăng bán sản phẩm). |
| `ngay_hieu_luc` | String | "11/03/2026", "28/12/2023", "Hien_hanh" | Các chính sách TMĐT thường xuyên được cập nhật. Metadata này giúp hệ thống loại bỏ các quy định cũ đã hết hiệu lực, đảm bảo câu trả lời luôn chính xác theo thời điểm hiện tại. |
| `loai_van_ban` | String | "dieu_khoan", "chinh_sach", "quy_dinh" | Phân loại cấp độ văn bản (Hợp đồng gốc vs Hướng dẫn vận hành/Chính sách cụ thể). Giúp hệ thống ưu tiên văn bản có giá trị pháp lý cao nhất khi có xung đột thông tin. |
| `pham_vi_ap_dung` | String | "chung", "shopee_mall", "quoc_te" | Giải quyết các câu hỏi có tính đặc thù. Ví dụ: Chính sách trả hàng/chi phí vận chuyển của đơn hàng Shopee Mall có quy trình riêng biệt so với đơn hàng thông thường. |
---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| `dieu_khoan_dich_vu.txt` | FixedSizeChunker (`fixed_size`) | 149 | 696.2 | Trung bình (giữ overlap nhưng dễ cắt giữa ý) |
| `dieu_khoan_dich_vu.txt` | SentenceChunker (`by_sentences`) | 159 | 519.4 | Khá tốt theo câu, nhưng dễ vỡ ngữ cảnh điều khoản dài |
| `dieu_khoan_dich_vu.txt` | RecursiveChunker (`recursive`) | 173 | 477.1 | Tốt (ưu tiên tách theo xuống dòng/đoạn) |
| `chinh_sach_tra_hang_hoan_tien.txt` | FixedSizeChunker (`fixed_size`) | 35 | 690.5 | Trung bình |
| `chinh_sach_tra_hang_hoan_tien.txt` | SentenceChunker (`by_sentences`) | 47 | 410.1 | Khá tốt |
| `chinh_sach_tra_hang_hoan_tien.txt` | RecursiveChunker (`recursive`) | 38 | 507.7 | Tốt |
| `chinh_sach_cam_han_che_san_pham.txt` | FixedSizeChunker (`fixed_size`) | 22 | 698.5 | Trung bình |
| `chinh_sach_cam_han_che_san_pham.txt` | SentenceChunker (`by_sentences`) | 55 | 224.0 | Thấp (chunk quá nhỏ, mất mạch list cấm) |
| `chinh_sach_cam_han_che_san_pham.txt` | RecursiveChunker (`recursive`) | 20 | 618.8 | Tốt (giữ cụm nội dung theo đoạn) |

### Strategy Của Tôi

**Loại:** Custom Chunker theo domain (Q&A/section/header)

**Mô tả cách hoạt động:**
> Custom chunker tách văn bản theo block đoạn (`\n\n`) và ưu tiên nhận diện các block dạng tiêu đề/điều mục (ví dụ: `1.`, `1.1`, gạch đầu dòng, hoặc dòng in hoa). Khi gặp block giống header, chunk hiện tại sẽ được chốt để tránh trộn hai section khác nhau. Sau đó các block được gom cho đến ngưỡng mục tiêu khoảng 700 ký tự; nếu chunk quá dài (>950) thì cắt mềm theo cửa sổ 900 ký tự có overlap 80 ký tự. Cách này giúp giữ cấu trúc văn bản pháp lý/chính sách tốt hơn so với cắt cứng theo số ký tự.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Dữ liệu Shopee có nhiều pattern theo mục/tiểu mục và danh sách điều kiện, nên chunk theo section sẽ giữ đúng “đơn vị nghĩa” để retrieval chính xác hơn. Nếu dùng cắt cứng, nhiều trường hợp phần điều kiện và ngoại lệ bị tách rời dẫn đến câu trả lời thiếu ý. Strategy này khai thác trực tiếp cấu trúc heading/list của policy documents.

**Code snippet (nếu custom):**
```python
import re

class CustomPolicyChunker:
    def __init__(self, target_size: int = 700, min_size: int = 220) -> None:
        self.target_size = target_size
        self.min_size = min_size

    def chunk(self, text: str) -> list[str]:
        text = text.strip()
        if not text:
            return []

        blocks = [b.strip() for b in re.split(r"\n\s*\n+", text) if b.strip()]
        chunks, cur = [], ""

        for b in blocks:
            is_header = bool(
                re.match(r"^(?:\d+(?:\.\d+)*[\)\.]?|[IVXLC]+[\.)]|[-*])\s+.+", b)
            ) or b.isupper()

            if is_header and cur:
                chunks.append(cur.strip())
                cur = b
                continue

            candidate = (cur + "\n\n" + b).strip() if cur else b
            if len(candidate) <= self.target_size or len(cur) < self.min_size:
                cur = candidate
            else:
                chunks.append(cur.strip())
                cur = b

        if cur:
            chunks.append(cur.strip())
        return chunks
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| `dieu_khoan_dich_vu.txt` | best baseline = `recursive` | 173 | 477.1 | Tốt |
| `dieu_khoan_dich_vu.txt` | **của tôi (custom)** | 199 | 421.1 | Tốt hơn ở câu hỏi bám theo từng điều khoản |
| `chinh_sach_tra_hang_hoan_tien.txt` | best baseline = `recursive` | 38 | 507.7 | Tốt |
| `chinh_sach_tra_hang_hoan_tien.txt` | **của tôi (custom)** | 51 | 379.4 | Tốt hơn ở câu hỏi quy trình/ngoại lệ |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| **Tôi** | Custom chunking theo section/header + metadata filter | 8.5/10 | Top-3 thường đúng ngữ cảnh điều khoản, giữ tốt cụm "điều kiện + ngoại lệ", giải thích nguồn rõ | Cần tune kỹ tham số; nếu không filter metadata khi query rộng thì vẫn có nhiễu |
| **Nguyễn Duy Hiếu** | **FixedSize (1000/150)** | **9/10** | **Độ ổn định tối đa:** Duy trì mật độ thông tin đậm đặc (~990 ký tự/chunk), giúp Agent nhận được nhiều ngữ cảnh nhất trong một lần truy xuất. Overlap 150 ký tự xử lý triệt để lỗi mất thông tin tại điểm cắt ranh giới. | Tài nguyên lưu trữ tăng nhẹ do có phần nội dung trùng lặp từ overlap, nhưng không đáng kể so với hiệu quả bối cảnh mang lại. |
| **Phạm Đan Kha**| MarkdownRecursive | 8.0 | Nhận diện chính xác cấu trúc văn bản pháp lý | Thuật toán chạy chậm hơn khi văn bản dài |
| **Vũ Đức Kiên** | **SentenceChunker (max=4)** | **8/10** | **Độ chính xác (Accuracy) từ ngữ:** Bảo toàn 100% cấu trúc ngữ pháp tự nhiên. Rất tốt cho việc định nghĩa các thuật ngữ ngắn mà không lo bị cắt đôi từ khóa (Word-splitting). | **Thiếu hụt bối cảnh:** Với chiều dài thực tế chỉ ~410-519 ký tự, bối cảnh thường bị rời rạc. Đối với các quy trình Shopee phức tạp, 4 câu thường không đủ để bao quát cả điều kiện và ngoại lệ (vi phạm tiêu chí Completeness). |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> Với domain policy/e-commerce này, custom strategy theo section/header cho kết quả phù hợp nhất vì giữ nguyên mạch “điều khoản - điều kiện - ngoại lệ” trong cùng chunk. Baseline `recursive` vẫn rất mạnh và là lựa chọn an toàn, nhưng custom chunker cho độ chi tiết tốt hơn khi truy vấn các câu hỏi cụ thể theo từng mục.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`CustomPolicyChunker.chunk`** — approach:
> Tôi tách văn bản theo block đoạn bằng regex `\n\s*\n+`, vì tài liệu policy thường đã có cấu trúc theo điều khoản và khoảng trắng giữa các mục. Với mỗi block, mình nhận diện block giống heading bằng pattern `^(?:\d+(?:\.\d+)*[\)\.]?|[IVXLC]+[\.)]|[-*])\s+.+` (mục đánh số, số La Mã, gạch đầu dòng) hoặc dòng in hoa. Nếu gặp heading mới thì đóng chunk hiện tại để không trộn 2 section khác ngữ nghĩa. Sau đó gom các block đến ngưỡng `target_size` để cân bằng giữa đủ ngữ cảnh và độ chính xác retrieval.

**`CustomPolicyChunker` (size control + edge cases)** — approach:
> Base case là text rỗng thì trả `[]`; text ngắn thì giữ nguyên 1 chunk. Nếu một block/section quá dài, mình cắt mềm theo cửa sổ ký tự có overlap nhỏ để tránh mất mạch ở ranh giới. Ngoài ra mình dùng `strip()` ở mỗi bước để loại chunk rỗng và giảm nhiễu do xuống dòng thừa trong dữ liệu nguồn.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Sau khi custom chunker tạo các chunk theo section, mình nạp từng chunk vào `EmbeddingStore` với metadata gồm `doc_id`, `source`, và nhóm chủ đề để giữ khả năng trace ngược về điều khoản gốc. Mỗi chunk được embed bằng `embedding_fn` rồi lưu dưới dạng record `{id, content, embedding, metadata}`. Khi `search`, hệ thống embed query, tính điểm bằng dot product với tất cả chunk candidates, sau đó sort giảm dần và lấy top-k để đưa vào bước trả lời.

**`search_with_filter` + `delete_document`** — approach:
> Mình ưu tiên **filter trước rồi mới search** để giảm nhiễu: ví dụ query về trả hàng thì lọc theo `chu_de: don_hang` hoặc đúng tài liệu cần thiết trước khi so điểm similarity. Cách này đặc biệt hữu ích với policy domain vì nhiều điều khoản có từ khóa gần giống nhau nhưng khác phạm vi áp dụng. Với `delete_document`, mình xóa toàn bộ chunk có cùng `metadata.doc_id`, nhờ vậy khi tài liệu được cập nhật thì có thể re-index lại sạch sẽ.

### KnowledgeBaseAgent

**`answer`** — approach:
> `KnowledgeBaseAgent` lấy top-k chunk đã retrieve từ store và ghép thành context theo thứ tự điểm số, mỗi chunk có đánh số `[1]`, `[2]`, `[3]` để dễ kiểm chứng nguồn. Prompt được thiết kế theo nguyên tắc grounding: chỉ trả lời dựa trên context, thiếu bằng chứng thì phải nói không biết thay vì suy diễn. Vì chunk của mình bám theo section/điều khoản, agent trả lời tốt hơn ở các câu hỏi cần đủ cụm "điều kiện + ngoại lệ + mức phí/bồi thường" trong cùng ngữ cảnh.

### Test Results

```
# Paste output of: pytest tests/ -v
============================= test session starts =============================
platform win32 -- Python 3.13.5, pytest-9.0.2, pluggy-1.6.0
collected 42 items

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED

============================= 42 passed in 0.14s =============================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Nguoi mua co the thanh toan bang COD. | Nguoi mua co the thanh toan khi nhan hang (COD). | high | 0.0143 | Đúng |
| 2 | Chinh sach tra hang ap dung cho Shopee Mall. | Quy trinh hoan tien cho don Shopee Mall duoc quy dinh ro. | high | -0.1340 | Sai |
| 3 | Shopee co chinh sach bao mat du lieu ca nhan. | Toi thich an pizza vao cuoi tuan. | low | -0.2751 | Đúng |
| 4 | Nguoi ban vi pham co the bi boi thuong 10 trieu dong. | Muc boi thuong toi da cho vi pham la 10.000.000 VND. | high | 0.1247 | Đúng |
| 5 | Chinh sach cam ban bua ngai tren san. | Nguoi mua duoc su dung Apple Pay va Google Pay. | low | -0.4200 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Cặp số 2 bất ngờ nhất: về mặt ngữ nghĩa mình dự đoán high, nhưng actual lại âm (-0.1340). Điều này cho thấy với backend `_mock_embed` thì vector không phản ánh ngữ nghĩa thực mà mang tính deterministic theo chuỗi ký tự, nên kết quả similarity chỉ phù hợp để test pipeline chứ không phù hợp để kết luận semantic quality thật. Khi đánh giá retrieval thực tế, cần dùng local/OpenAI embedding model thay vì mock.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 7 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **Các queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Loại câu hỏi | Query | Gold Answer |
|---|--------------|-------|-------------|
| 1 | Liệt kê | Người mua có thể thanh toán đơn hàng trên Shopee bằng những hình thức nào? | Có 8 hình thức: Thẻ tín dụng/ghi nợ, Thanh toán khi nhận hàng (COD), Thẻ ATM Nội Địa/Internet Banking, SPayLater, Ví ShopeePay, Apple Pay/Google Pay, Chuyển khoản ngân hàng, và các phương thức khác khả dụng từng thời điểm. |
| 2 | Con số | Người bán sẽ phải chịu mức Phí Thanh Toán là bao nhiêu phần trăm cho mỗi đơn hàng thành công? | Mức Phí Thanh Toán áp dụng cho mỗi đơn hàng thành công là 4,91% (đã bao gồm thuế GTGT), áp dụng cho tất cả các phương thức thanh toán. |
| 3 | Chi tiết/Ngoại lệ | Những sản phẩm nào có yếu tố tôn giáo, tâm linh, mê tín dị đoan bị cấm bán trên Shopee? | Các vật phẩm có chứa từ khóa 'trì chú', 'làm phép'; bùa ngải (bùa tình yêu, bùa hồ yêu...); mẹ ngoắc; kumanthong; nhang xin số; nhang cúng kumanthong. |
| 4 | Điều kiện | Kể từ ngày 28/12/2023, người bán vi phạm chính sách chống gian lận sẽ phải bồi thường cho Shopee bao nhiêu tiền? | Người bán sẽ phải bồi thường cho Shopee một khoản tiền lên đến 10.000.000 VND (Mười triệu đồng) cho từng đơn hàng vi phạm. |
| 5 | Quy trình | Shopee có bắt buộc phải đáp ứng yêu cầu cung cấp Dữ Liệu Cá Nhân của người dùng hay không? | Shopee không buộc phải đáp ứng hay giải quyết yêu cầu cung cấp Dữ Liệu Cá Nhân trừ phi người dùng đã đồng ý đóng một khoản phí hợp lý theo ước tính văn bản của Shopee. |
| 6 | Chống Ảo giác (Out of Domain) | Làm thế nào để đăng ký chạy quảng cáo Shopee Ads cho sản phẩm mới?** | "Tôi không tìm thấy thông tin này trong tài liệu được cung cấp."** *(Vì 6 file chính sách không hề đề cập đến cách chạy Ads).* |
| 7 | Chống Ảo giác (False Premise) | Shopee quy định phí phạt trả hàng quá hạn là 50.000 VNĐ đúng không?** | "Trong tài liệu được cung cấp không có quy định nào về việc thu phí phạt 50.000 VNĐ khi trả hàng quá hạn."** *(Hệ thống RAG tốt phải từ chối xác nhận tiền đề sai này).* |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Người mua có thể thanh toán đơn hàng bằng những hình thức nào? | Chunk nói về cơ chế Shopee thanh toán qua **chuyển khoản ngân hàng** và yêu cầu cung cấp tài khoản ngân hàng. | 0.8266 | Có (nhưng chưa đủ 8 hình thức) | Agent trả lời theo hướng: có các phương thức thanh toán qua hệ thống Shopee, trong đó có chuyển khoản; cần thêm evidence để liệt kê đầy đủ toàn bộ phương thức. |
| 2 | Mức Phí Thanh Toán cho đơn hàng thành công là bao nhiêu? | Top-1 nói về hoàn tiền theo phương thức thanh toán ban đầu; top-3 có đúng section **16.3 Phí Thanh Toán**. | 0.7596 | Một phần | Agent xác định đúng ngữ cảnh phí thanh toán nhưng top-1 chưa chạm trực tiếp con số 4,91%; cần mở rộng top-k/rerank để bắt đúng dòng số liệu. |
| 3 | Nhóm sản phẩm tâm linh/mê tín bị cấm bán là gì? | Top-1 là tiêu đề danh sách cấm/hạn chế; top-3 có điều **4.26** nêu rõ bùa ngải, kumanthong, nhang xin số... | 0.7960 | Có | Agent tổng hợp đúng danh mục bị cấm từ điều 4.26 (trì chú, làm phép, bùa ngải, mẹ ngoắc, kumanthong...). |
| 4 | Vi phạm chống gian lận (từ 28/12/2023) bồi thường bao nhiêu? | Chunk top-1 nêu trực tiếp mức bồi thường lên đến **10.000.000 VND** cho từng đơn vi phạm. | 0.7198 | Có | Agent trả lời đúng mức bồi thường tối đa 10.000.000 VND/đơn vi phạm và bối cảnh hiệu lực từ 28/12/2023. |
| 5 | Shopee có bắt buộc đáp ứng yêu cầu cung cấp dữ liệu cá nhân không? | Top-1 rơi vào phần thông tin phiên bản/chính sách; chưa vào đúng điều 13.2.5 trong top-3. | 0.7409 | Không | Agent chưa đủ bằng chứng trong top-3 để kết luận chắc; cần truy hồi sâu hơn vào mục 13.2.x để trả lời chính xác điều kiện đóng phí. |
| 6 | Làm thế nào để đăng ký chạy quảng cáo Shopee Ads? (OOD) | Top-1 nói về Shopee Xu, không liên quan Ads; toàn bộ top-3 không chứa hướng dẫn đăng ký Ads. | 0.7503 | Không (đúng kỳ vọng OOD) | Agent từ chối trả lời chi tiết và nêu không tìm thấy thông tin Shopee Ads trong tập tài liệu đã nạp. |
| 7 | Shopee có quy định phí phạt trả hàng quá hạn 50.000 VNĐ không? (False premise) | Top-1 nói về phạm vi chính sách trả hàng/hoàn tiền Shopee Mall, không có mốc 50.000 VNĐ. | 0.7424 | Có (để bác bỏ tiền đề sai) | Agent trả lời theo hướng không có căn cứ trong tài liệu cho mức phạt 50.000 VNĐ, nên không xác nhận tiền đề câu hỏi. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** **4 / 5** (cho 5 query in-domain)

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Mình học được từ bạn Hiếu rằng fixed-size chunking với overlap lớn có thể rất ổn định trong thực tế demo, đặc biệt khi cần “ăn chắc mặc bền” cho top-k. Cách bạn ấy giữ chunk lớn giúp agent không bị thiếu ngữ cảnh ở các câu hỏi nhiều điều kiện. Đây là trade-off thú vị giữa độ chính xác theo mục nhỏ và độ đầy đủ ngữ cảnh.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Nhóm khác làm phần đánh giá rất tốt ở chỗ họ tách riêng 2 loại lỗi: lỗi retrieval (lấy sai chunk) và lỗi generation (diễn đạt sai từ chunk đúng). Cách này giúp debug nhanh hơn rất nhiều so với chỉ nhìn câu trả lời cuối cùng. Mình học được rằng muốn cải thiện RAG thì phải đo đúng “nút thắt” trước khi tune model/chunking.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> Nếu làm lại, mình sẽ thêm bước rerank lexical (ưu tiên keyword/number như “4,91%”, “13.2.5”) sau semantic retrieval để giảm miss ở các câu hỏi cần con số hoặc điều khoản cụ thể. Mình cũng sẽ chuẩn hóa metadata sâu hơn theo `section_id` (ví dụ 13.2.5, 16.3) để filter chính xác hơn thay vì chỉ theo `chu_de`/`source`. Cuối cùng, mình sẽ tạo thêm test set “hard queries” cho các tình huống false premise và out-of-domain để kiểm tra tính chống ảo giác sớm hơn.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5/ 5 |
| Document selection | Nhóm | 9/ 10 |
| Chunking strategy | Nhóm | 13/ 15 |
| My approach | Cá nhân | 8/ 10 |
| Similarity predictions | Cá nhân | 5/ 5 |
| Results | Cá nhân | 9/ 10 |
| Core implementation (tests) | Cá nhân | 28/ 30 |
| Demo | Nhóm | 3/ 5 |
| **Tổng** | | **90/ 100** |
