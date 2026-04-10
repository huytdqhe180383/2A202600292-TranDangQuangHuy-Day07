# GUIDE: Chunking Strategies for Shopee Policy Retrieval

## 1) Muc tieu cua guide

Tai lieu nay giup:
- Hieu ro 4 loai chunking strategy dang dung trong lab.
- Biet khi nao nen chon strategy nao cho domain policy/e-commerce.
- Co kich ban trinh bay custom strategy truoc lop mot cach thuyet phuc.

Phan 4 (Custom Policy Chunker) la phan trong tam de pitch.

---

## 2) Tong quan 4 strategies

### A. FixedSizeChunker (`fixed_size`)

**Y tuong**
- Cat text theo kich thuoc co dinh (`chunk_size`) va co overlap (`overlap`).
- Vi du: `chunk_size=700`, `overlap=100` -> moi chunk moi se chia se 100 ky tu voi chunk truoc.

**Uu diem**
- Don gian, de du doan.
- Toc do nhanh, tinh on dinh cao.
- Co overlap nen giam mat context o ranh gioi.

**Nhuoc diem**
- Cat "mu" theo ky tu, de cat ngang cau/ngu canh.
- De tron nhieu y khong lien quan vao cung mot chunk (neu size lon).

**Khi nen dung**
- Baseline dau tien.
- Du lieu van ban khong co cau truc ro rang.
- Muon co benchmark de so sanh voi cac strategy khac.

---

### B. SentenceChunker (`by_sentences`)

**Y tuong**
- Tach theo ranh gioi cau (., !, ?) roi gom N cau vao mot chunk.
- Tham so chinh: `max_sentences_per_chunk`.

**Uu diem**
- Chunk de doc, de hieu.
- Han che cat ngang cau.

**Nhuoc diem**
- Voi van ban policy dai, 1 y thuong trai qua nhieu cau -> de vo ngu canh.
- Danh sach ngoai le/dieu kien de bi tach qua nho.

**Khi nen dung**
- Van ban mang tinh mo ta ngan, y ro tung cau.
- Query hoi thong tin ngan, truc tiep.

---

### C. RecursiveChunker (`recursive`)

**Y tuong**
- Tach theo thu tu uu tien separator: `\n\n`, `\n`, `. `, ` `, roi moi fallback fixed-size.
- Muc tieu: uu tien cat theo cau truc tu nhien truoc khi cat cung.

**Uu diem**
- Thuong giu context tot hon fixed-size.
- Can bang kha tot giua coherence va chunk size.
- La baseline "manh", rat hop cho policy text.

**Nhuoc diem**
- Van la generic strategy, chua "biet domain".
- Khong tu hieu section heading cua tai lieu chinh sach.

**Khi nen dung**
- Muon ket qua retrieval tot ma chua co custom strategy.
- Du lieu co xuong dong/doan tuong doi ro.

---

### D. CustomPolicyChunker (`custom_policy`)

**Y tuong**
- Domain-aware chunking cho policy docs:
  - Tach theo block doan (`\n\n`).
  - Nhan dien heading/section (so muc, so La Ma, bullet, dong in hoa).
  - Dong chunk khi gap heading moi de tranh tron 2 section khac nhau.
  - Gom block den `target_size`, neu qua dai thi split mem co overlap nho.

**Uu diem**
- Giu "don vi nghia" theo dieu khoan.
- Tot hon cho query co dieu kien/ngoai le/phi muc cu the.
- De trace nguon khi giai thich answer.

**Nhuoc diem**
- Can tune ky hon (target_size, min_size, hard_limit).
- Phu thuoc pattern dinh dang cua tai lieu.

**Khi nen dung**
- Domain policy, legal, SOP, docs co heading/section ro.
- Khi can demo retrieval precision va grounding quality.

---

## 3) Bang so sanh nhanh

| Strategy | Coherence | Precision query cu the | Do on dinh | Do de tune | Hop policy domain |
|---|---|---|---|---|---|
| fixed_size | Trung binh | Trung binh-thap | Cao | De | Trung binh |
| by_sentences | Kha | Trung binh | Kha | De | Trung binh-thap |
| recursive | Tot | Kha-tot | Kha cao | Vua | Tot |
| custom_policy | Rat tot (neu tune dung) | Tot-rat tot | Vua | Kho hon | Rat tot |

---

## 4) Huong dan tune params (thuc chien)

### 4.1 Tham so quan trong

- `chunk_size`:
  - Nho hon -> precision cao hon, nhung de mat context.
  - Lon hon -> context day hon, nhung de loang y.
- `overlap` (fixed-size):
  - Tang overlap -> giu context ranh gioi tot hon, doi lai tang so chunk.
- `max_sentences_per_chunk`:
  - 2-4 thuong hop ly; qua nho de vo y, qua lon de loang.
- `custom min_size`:
  - Tranh tao qua nhieu chunk qua ngan.
- `custom hard_limit`:
  - Tran tren de mot chunk khong qua dai.
- `top_k`:
  - De demo nen de 3-5; 5 de de bat evidence hon.

### 4.2 Preset de bat dau (khuyen nghi)

- Strategy: `custom_policy` hoac `recursive`
- `chunk_size`: 500-650
- `custom min_size`: 180-240
- `custom hard_limit`: 700-850
- `top_k`: 5
- Metadata filter: uu tien `source` hoac `category` truoc khi search toan bo

### 4.3 Trieu chung va cach sua nhanh

- **Top-1 sai file** -> bat `source/category` filter, giam `chunk_size`.
- **Top-3 nhieu chunk nhieu nhieu, khong dung y** -> giam `hard_limit`, giam `min_size`.
- **Answer thieu dieu kien ngoai le** -> tang nhe `chunk_size` hoac `top_k`.
- **Score sat nhau** -> query qua mo ho; viet query cu the hon + dung metadata filter.

---

## 5) Pitch Custom Chunking (chi tiet de trinh bay truoc lop)

### 5.1 Problem statement (Mo dau 30-45s)

"Policy documents cua Shopee dai, nhieu dieu khoan va ngoai le. Neu cat text theo kich thuoc co dinh, chunk de cat ngang phan quan trong, lam retrieval lay sai context. Nhom em muon retrieval khong chi dung tu khoa, ma dung dung *section nghia*."

### 5.2 Design principle (Nguyen tac thiet ke)

1. **Ton trong cau truc van ban**  
   Heading va section la "ranh gioi nghia" quan trong hon ranh gioi ky tu.

2. **Can bang context va precision**  
   Chunk can du lon de giu dieu kien + ngoai le, nhung khong qua lon den muc loang.

3. **Grounding-friendly**  
   Chunk phai de truy vet nguon (source + doc_id + section gan dung) de agent tra loi co bang chung.

### 5.3 How it works (Co che hoat dong)

1. Split theo block doan (`\n\n`).
2. Detect heading-like blocks:
   - Mau danh so: `1.`, `1.1`, `I.`, bullet `-`, `*`
   - Dong in hoa toan bo.
3. Neu gap heading moi:
   - Dong chunk hien tai.
   - Bat dau chunk moi tu heading do.
4. Gom cac block lien quan den nguong `target_size`.
5. Neu chunk qua dai > `hard_limit`:
   - Split mem theo cua so nho hon co overlap.

### 5.4 Vi sao no hop domain Shopee policy

- Tai lieu chinh sach co cau truc section ro.
- Nhieu query can full "bo 3":
  - dieu kien ap dung
  - ngoai le
  - muc phi/boi thuong
- Custom chunking giup 3 thanh phan nay o cung vung context, retrieval de nhat quan hon.

### 5.5 Evidence framework de thuyet phuc

Khi trinh bay, dung 1 query co intent ro rang (vi du: "hinh thuc thanh toan"):

- Chay Panel A = baseline (`recursive`, no filter).
- Chay Panel B = `custom_policy` + filter `source=dieu_khoan_dich_vu.txt`.
- So sanh:
  - Top-1 co dung file khong.
  - Trong top-3 co bao nhieu chunk relevant.
  - Chunk co chua full thong tin lien quan khong.

Neu Panel B co:
- score top-1 on dinh hon,
- it chunk lac de hon,
- answer day du hon theo gold,
=> pitch: custom chunking tao loi the retrieval thuc te.

### 5.6 Script thuyet trinh mau (1-2 phut)

"Nhom em de bai toan retrieval policy ve dung don vi nghia thay vi don vi ky tu. Baseline recursive da tot, nhung van generic. CustomPolicyChunker cua nhom em nhan dien heading, khoa theo section, va gioi han do dai chunk de giu du context ma van sac net.  
Ket qua tren cung query cho thay khi bat metadata filter va dung custom chunking, top-k it nhieu hon, dung file hon, va evidence ro hon. Nghia la chat luong retrieval tang khong chi o score, ma o kha nang giai thich va grounding."

---

## 6) Goi y demo voi giao dien UI A/B

1. Chon cung tap file cho A va B.
2. Dung cung query va cung `top_k`.
3. A de baseline (`recursive`), B de `custom_policy`.
4. Bat metadata filter cho B (source hoac category).
5. So sánh truc tiep top-k score, source, va noi dung chunk.

Tip: co the chup man hinh ket qua A/B va dua vao slide.

---

## 7) Ket luan ngan

- Khong co strategy "best" cho moi domain.
- Voi policy docs, chunking theo section + metadata filter thuong la cap doi hieu qua nhat.
- Muc tieu cuoi cung khong chi la retrieve "co ve dung", ma la retrieve dung de agent tra loi co the kiem chung.
