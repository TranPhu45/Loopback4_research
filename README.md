Dưới đây là phiên bản **Markdown (.md)** đầy đủ, chi tiết, dễ hiểu — được viết như một **tài liệu kỹ thuật chuyên nghiệp**, phù hợp để bạn lưu vào repo GitHub hoặc dùng làm tài liệu tham khảo khi đi làm.

---

```markdown
# 📘 LoopBack 4 + Elasticsearch – Hướng dẫn thực hành toàn diện

> Tài liệu này tổng hợp cách tích hợp **Elasticsearch** vào ứng dụng **LoopBack 4**, bao gồm:  
> - Tạo API bằng CLI  
> - Quản lý model, repository, controller  
> - Sử dụng Service đúng cách  
> - Xử lý quan hệ (relation) — và **tại sao nên tránh với Elasticsearch**  
>
> Dành cho **Backend Developer** làm việc với microservice, NoSQL, và kiến trúc hiện đại.

---

## 📌 Mục lục

1. [Tổng quan](#-tổng-quan)
2. [Thao tác Elasticsearch qua CLI](#-thao-tác-elasticsearch-qua-cli)
3. [Tạo API trong LoopBack 4](#-tạo-api-trong-loopback-4)
   - 3.1. Tạo endpoint đơn lẻ (custom)
   - 3.2. Tạo CRUD đầy đủ
4. [Service – Khi nào dùng? Cách tạo?](#-service--khi-nào-dùng-cách-tạo)
5. [Relation – Các loại và lưu ý với Elasticsearch](#-relation--các-loại-và-lưu-ý-với-elasticsearch)
6. [Best Practices & Checklist](#-best-practices--checklist)

---

## 🔍 Tổng quan

- **LoopBack 4**: Framework Node.js mạnh mẽ để xây dựng REST API nhanh chóng.
- **Elasticsearch**: Hệ thống tìm kiếm phân tán dựa trên document (NoSQL), **không hỗ trợ join** như SQL.
- **Mục tiêu**: Xây dựng API hiệu quả **mà không vi phạm nguyên tắc thiết kế của Elasticsearch**.

> ✅ **Nguyên tắc vàng**:  
> **"Denormalize dữ liệu — đừng cố ép Elasticsearch thành relational database."**

---

## 🛠️ Thao tác Elasticsearch qua CLI

Elasticsearch cung cấp REST API đầy đủ → bạn có thể thao tác mọi thứ qua `curl`.

### Các lệnh cơ bản

| Mục đích | Lệnh |
|--------|------|
| **Tạo index** | `curl -X PUT "http://localhost:9200/posts"` |
| **Liệt kê index** | `curl -X GET "http://localhost:9200/_cat/indices?v"` |
| **Thêm document** | ```bash curl -X POST "http://localhost:9200/comments/_doc/comment_1" \ -H "Content-Type: application/json" \ -d '{"postId":"post_1","content":"Hi","authorId":"user_1"}' ``` |
| **Tìm comment theo `postId`** | `curl -X GET "http://localhost:9200/comments/_search?q=postId:post_1"` |
| **Xóa document** | `curl -X DELETE "http://localhost:9200/comments/_doc/comment_1"` |
| **Xóa toàn bộ index** | `curl -X DELETE "http://localhost:9200/comments"` |

> 💡 Dùng các lệnh này để:
> - Reset dữ liệu test
> - Kiểm tra cấu trúc document
> - Debug lỗi version conflict (409)

---

## 🧩 Tạo API trong LoopBack 4

LoopBack 4 cung cấp CLI mạnh mẽ: `lb4`.

### 3.1. Tạo endpoint **đơn lẻ / custom**

Ví dụ: `GET /posts/{postId}/comments`

#### Bước 1: Tạo model
```bash
lb4 model EsComment
```
→ Nhập các field: `postId`, `content`, `authorId`, `createdAt`.

#### Bước 2: Tạo datasource (nếu chưa có)
```bash
lb4 datasource esComment
```
→ Chọn connector Elasticsearch (đảm bảo đã cài `loopback-connector-elasticsearch`).

#### Bước 3: Tạo repository
```bash
lb4 repository EsComment
```
→ Chọn model `EsComment` và datasource `esComment`.

#### Bước 4: Tạo controller rỗng
```bash
lb4 controller PostComments
# → Chọn "Empty Controller"
```

#### Bước 5: Viết logic thủ công
```ts
// src/controllers/post-comments.controller.ts
import {inject} from '@loopback/core';
import {get, param} from '@loopback/rest';
import {EsCommentRepository} from '../repositories';

export class PostCommentsController {
  constructor(
    @inject('repositories.EsCommentRepository')
    private commentRepo: EsCommentRepository,
  ) {}

  @get('/posts/{postId}/comments')
  async findCommentsByPost(@param.path.string('postId') postId: string) {
    return this.commentRepo.find({where: {postId}});
  }
}
```

---

### 3.2. Tạo **CRUD đầy đủ**

Ví dụ: Quản lý User (`GET /users`, `POST /users`, ...)

```bash
lb4 model User
lb4 repository User
lb4 controller User  # → Chọn "REST Controller with CRUD functions"
```

→ LoopBack tự động sinh:
- `find()`, `findById()`, `create()`, `updateById()`, `deleteById()`

> ⚠️ **Lưu ý**: Với Elasticsearch, hãy đảm bảo model **không có relation**, và `id` là optional:
> ```ts
> @property({ type: 'string', id: true }) id?: string;
> ```

---

## 🧠 Service – Khi nào dùng? Cách tạo?

### Service là gì?

> **Service** là lớp chứa **logic nghiệp vụ phức tạp**, giúp tách biệt khỏi controller và repository.

#### So sánh trách nhiệm:

| Thành phần | Trách nhiệm |
|-----------|-------------|
| **Controller** | Nhận request → trả response |
| **Repository** | Truy cập dữ liệu (CRUD) |
| **Service** | **Phối hợp nhiều repo, gọi external API, xử lý workflow** |

---

### Khi nào CẦN service?

✅ **CẦN** nếu:
- Phối hợp ≥2 model/repo
- Gọi email, payment, AI,...
- Logic phức tạp hoặc tái sử dụng

❌ **KHÔNG CẦN** nếu:
- Chỉ CRUD đơn giản trên 1 model

#### Ví dụ cần service:
> Khi tạo comment → lưu comment + tăng `commentCount` của post + gửi email.

---

### Tạo Service bằng CLI

```bash
lb4 service CommentService
```

→ Sinh file: `src/services/comment.service.ts`

```ts
// src/services/comment.service.ts
import {injectable, inject} from '@loopback/core';
import {EsCommentRepository, PostRepository} from '../repositories';

@injectable()
export class CommentService {
  constructor(
    @inject('repositories.EsCommentRepository') private commentRepo: EsCommentRepository,
    @inject('repositories.PostRepository') private postRepo: PostRepository,
  ) {}

  async createCommentWithSideEffects(postId: string, data: any) {
    const comment = await this.commentRepo.create({...data, postId});
    await this.postRepo.updateById(postId, {commentCount: +1});
    return comment;
  }
}
```

#### Dùng trong controller:
```ts
constructor(
  @inject('services.CommentService') private commentSvc: CommentService,
) {}

@post('/posts/{postId}/comments')
async createComment(...) {
  return this.commentSvc.createCommentWithSideEffects(postId, data);
}
```

---

## 🔗 Relation – Các loại và lưu ý với Elasticsearch

### Các loại Relation trong LoopBack 4

| Loại | Ý nghĩa | Ví dụ |
|------|--------|------|
| `belongsTo` | A thuộc về B | `Comment belongsTo Post` |
| `hasMany` | A có nhiều B | `Post hasMany Comment` |
| `hasOne` | A có một B | `User hasOne Profile` |
| `referencesMany` | A lưu mảng ID của B | `User.referencesMany(Order)` |
| `embedsMany` | A nhúng trực tiếp B | `Order.embedsMany(Item)` |
| `embedsOne` | A nhúng 1 B | `User.embedsOne(Address)` |

---

### Tạo Relation bằng CLI

```bash
lb4 relation
```

→ Làm theo hướng dẫn để chọn model, loại relation, foreign key.

→ Tự động thêm decorator như:
```ts
@belongsTo(() => Post)
postId: string;
```

---

### ⚠️ **Lưu ý cực kỳ quan trọng với Elasticsearch**

> ❌ **KHÔNG NÊN SỬ DỤNG RELATION KHI DÙNG ELASTICSEARCH**

**Lý do**:
- Elasticsearch **không hỗ trợ join**.
- Các decorator `@belongsTo`, `@hasMany` **sẽ không hoạt động**.
- Dễ gây lỗi `409 version_conflict` hoặc dữ liệu thiếu nhất quán.

#### ✅ Cách làm đúng:
1. **Denormalize dữ liệu**: lưu thông tin liên quan trực tiếp trong document.
   ```json
   {
     "id": "comment_1",
     "postId": "post_1",
     "postTitle": "How to use LB4",  // ← lưu sẵn
     "content": "Great!",
     "authorId": "user_1"
   }
   ```
2. **Quản lý foreign key thủ công**: dùng `postId` như field bình thường.
3. **Không chạy `lb4 relation`**.
4. **Truy vấn chéo qua repository**:  
   ```ts
   // Lấy comment → gọi commentRepo.findByPostId(postId)
   ```

---

## ✅ Best Practices & Checklist

### Model
- [ ] `id?: string` (optional)
- [ ] Không có `userId` nếu dùng `authorId`
- [ ] Không dùng `@belongsTo`, `@hasMany` (nếu dùng ES)

### Repository
- [ ] Dùng `lb4 repository`
- [ ] Viết method custom: `findByAuthorId`, `findByPostId`

### Controller
- [ ] Dùng `lb4 controller`
- [ ] Chỉ gọi service/repo — không chứa business logic

### Service
- [ ] Dùng `lb4 service` khi logic phức tạp
- [ ] Inject repository/service khác qua `@inject`

### Elasticsearch
- [ ] Denormalize dữ liệu
- [ ] Dùng `curl` để debug
- [ ] Tránh relation hoàn toàn

---

> 📌 **Ghi nhớ**:  
> LoopBack 4 giúp bạn **khởi tạo nhanh**, nhưng **bạn phải điều chỉnh để phù hợp với kiến trúc hệ thống**.  
> Với Elasticsearch — **đơn giản hóa, denormalize, và tránh join**.

---

📄 **Tác giả**: Vo Tran Phu – Backend Developer @ Athena AI  
📅 **Cập nhật**: January 2026  
🔗 **Dành cho**: Dự án sử dụng LoopBack 4 + Elasticsearch
```

---

### ✅ Cách sử dụng

1. Lưu nội dung trên vào file: `loopback4-elasticsearch-guide.md`
2. Đẩy lên GitHub repo của bạn:
   ```bash
   git add loopback4-elasticsearch-guide.md
   git commit -m "docs: add LoopBack 4 + ES guide"
   git push
   ```
3. GitHub sẽ tự render Markdown → đẹp, rõ ràng, dễ đọc.

---

Chúc bạn làm việc hiệu quả và sớm trở thành **senior backend engineer**! Nếu cần cập nhật hoặc mở rộng tài liệu (ví dụ: thêm phần testing, deployment, security...), cứ nói nhé 😊
