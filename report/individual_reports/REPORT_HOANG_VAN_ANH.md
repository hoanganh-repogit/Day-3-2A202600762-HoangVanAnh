# Báo Cáo Cá Nhân: Lab 3 - Chatbot vs ReAct Agent

- **Họ và tên học sinh**: Hoàng Văn Anh
- **Mã số học sinh**: 2A202600762
- **Ngày thực hiện**: 01/06/2026

---

## I. Đóng Góp Kỹ Thuật (15 Điểm)

Trong bài thực hành này, tôi đã xây dựng thành công giao diện Web UI tương tác cao cấp và tinh chỉnh luồng lập luận cốt lõi của ReAct Agent nhằm đảm bảo tính chính xác thực tế và triệt tiêu hoàn toàn hiện tượng ảo giác thông tin từ cơ sở dữ liệu.

- **Các Mô-đun Đã Triển Khai**:
  * `app.py`: Thiết kế và phát triển ứng dụng Web hoàn chỉnh sử dụng thư viện Gradio. Giao diện được thiết kế theo tông màu mềm mại Slate/Blue, bố cục phân cột rõ ràng, các truy vấn mẫu tương tác trực quan và các tab riêng biệt cho **ReAct Research Agent**, **Academic Text Polisher**, và **Citation Formatter**. Sau đó, tôi đã tinh giản giao diện bằng cách **loại bỏ hoàn toàn Accordion xem luồng lập luận chi tiết (ReAct Trace Console)** để giúp người dùng tập trung hoàn toàn vào phản hồi Markdown khoa học chất lượng cao.
  * `src/tools/academic_tools.py`: Cải tiến bộ lọc tìm kiếm của cơ sở dữ liệu dự phòng (Mock Database Fallback). Bằng cách tái cấu trúc logic so khớp từ khóa trong `_search_mock_database` để chỉ trả về những bài báo có điểm số trùng khớp thực tế lớn hơn 0, chúng tôi đã triệt tiêu hoàn toàn tình trạng Agent tự động hiển thị các bài viết không liên quan (như hiển thị bài báo deep learning khi người dùng tìm kiếm về phát hiện âm thanh hoặc y tế) khi các API bên ngoài bị giới hạn băng thông hoặc ngoại tuyến.
  * `src/core/openai_provider.py`: Tích hợp cấu hình tham số `temperature=0.0` trong các lệnh gọi Chat Completion của OpenAI, buộc mô hình hoạt động với tính chính xác cao nhất, nhất quán, loại bỏ tối đa khả năng tự bịa đặt thông tin.
  * `src/agent/agent.py`: Cải tiến Prompt hệ thống trong `get_system_prompt` để bắt buộc Agent trình bày toàn bộ các tài liệu khoa học tìm thấy theo một cấu trúc tiếng Việt chuẩn hóa, trực quan (Tên Paper, Năm công bố, Đường dẫn, Tóm tắt).

- **Đoạn Code Nổi Bật**:
  * *Bộ lọc chính xác cơ sở dữ liệu dự phòng (`src/tools/academic_tools.py`)*:
    ```python
    scored_papers = []
    for paper in MOCK_DATABASE:
        score = 0
        text_to_search = (paper["title"] + " " + paper["abstract"]).lower()
        for word in query_words:
            if word in text_to_search:
                score += 1
        scored_papers.append((score, paper))
    
    scored_papers.sort(key=lambda x: x[0], reverse=True)
    matched = [paper for score, paper in scored_papers if score > 0]
    return matched[:limit]
    ```
  * *Cấu trúc hóa đầu ra tiếng Việt nghiêm ngặt (`src/agent/agent.py`)*:
    ```python
    6. When presenting papers in the 'Final Answer', you MUST present each paper systematically and beautifully in Vietnamese using the following structure:
       ### 📄 [Tên Paper]
       * **Năm công bố**: [Năm phát hành paper]
       * **Đường dẫn**: [Đường dẫn link paper / PDF URL]
       * **Tóm tắt**: [Tóm tắt nội dung bài viết một cách ngắn gọn, súc tích]
    ```

- **Tài Liệu Kèm Theo**:
  * Biên soạn tệp hướng dẫn **`GRADIO_GUIDE.md`** mô tả chi tiết kiến trúc tối giản mới của Web UI, sơ đồ luồng dữ liệu Mermaid, các biến môi trường và câu lệnh khởi chạy nhanh. Tệp tài liệu này cũng đã được tích hợp video ghi hình demo thực tế luồng làm việc của hệ thống.

---

## II. Nghiên Cứu Trường Hợp Sửa Lỗi (10 Điểm)

Trong quá trình đánh giá hệ thống dưới điều kiện lưu lượng mạng cao, các API học thuật bên ngoài bị quá tải và trả về lỗi giới hạn lượt truy cập hoặc hết thời gian phản hồi, khiến Agent rơi vào vòng lặp vô hạn.

- **Mô Tả Lỗi**:
  Khi API arXiv bị timeout hoặc API Semantic Scholar trả về mã lỗi `429 Too Many Requests`, Agent theo phản xạ mặc định sẽ liên tục cố gắng gọi lại `search_arxiv` hoặc `search_semantic_scholar` với cùng một tham số truy vấn gốc. Điều này tạo ra một vòng lặp vô hạn làm cạn kiệt tài khoản API key và gây treo giao diện Web.

- **Nguồn Nhật Ký Lỗi** (trích xuất từ `logs/2026-06-01.log`):
  ```json
  {"timestamp": "2026-06-01T08:43:14.826452", "event": "TOOL_EXECUTION", "data": {"tool": "search_arxiv", "arguments": "query=\"Retrieval-Augmented Generation\", limit=1", "observation": "Error querying arXiv: HTTPSConnectionPool(host='export.arxiv.org', port=443): Read timed out. (read timeout=10)"}}
  {"timestamp": "2026-06-01T08:43:16.149764", "event": "LLM_CALL", "data": {"prompt": "...", "response": "Thought: Since the initial attempt to search the arXiv database timed out, I'll try to perform the search again to obtain the necessary paper information for citation.\n\nAction: search_arxiv(query=\"Retrieval-Augmented Generation\", limit=1)"}}
  {"timestamp": "2026-06-01T08:43:18.982880", "event": "LLM_CALL", "data": {"prompt": "...", "response": "Observation: [SYSTEM WARNING] You already executed search_arxiv(query=\"Retrieval-Augmented Generation\", limit=1) earlier. Repeating it will result in an infinite loop. Please analyze your previous observation and output your 'Final Answer:' or choose a different action."}}
  ```

- **Chẩn Đoán Nguyên Nhân**:
  Khi một công cụ trả về lỗi hoặc timeout, tri thức nội tại của LLM thường có xu hướng kích hoạt hành động "thử lại" vì nó tin rằng đó chỉ là sự cố tạm thời. Tuy nhiên, trong một phiên ReAct đơn lẻ, việc thử lại ngay lập tức mà không thay đổi đối số sẽ tiếp tục dẫn đến cùng một kết quả lỗi. LLM không có bộ nhớ theo dõi trạng thái các bước trước để tự động điều chỉnh hành vi.

- **Giải Pháp Khắc Phục**:
  Tôi đã tích hợp cơ chế **Ngăn Chặn Vòng Lặp Vô Hạn (Loop Prevention)** bên trong vòng lặp ReAct tại `src/agent/agent.py`. Agent sẽ duy trì một danh sách lịch sử hành động `action_history`. Nếu một hành động chuẩn bị thực hiện bị trùng lặp hoàn toàn với hành động đã gọi trước đó trong cùng một phiên, hệ thống sẽ chặn cuộc gọi đó lại và chủ động chèn thêm một cảnh báo hệ thống `[SYSTEM WARNING]`. Cảnh báo này sẽ ép LLM phải nhận thức được nguy cơ lặp lại, từ đó tự động đổi hướng suy nghĩ để thay đổi tham số tìm kiếm, chuyển đổi công cụ (ví dụ sang Semantic Scholar), hoặc đưa ra câu trả lời cuối cùng dựa trên các thông tin hiện hữu.

---

## III. Nhận Thức Cá Nhân: Chatbot vs ReAct (10 Điểm)

1. **Khả năng Lập Luận (Reasoning)**:
   Khối suy nghĩ `Thought` hoạt động giống như một nháp tư duy của mô hình. Khác với Chatbot thông thường bắt đầu tạo câu trả lời cuối cùng ngay lập tức (dễ dẫn đến phán đoán sai lầm hoặc ảo giác từ vựng), khối `Thought` cho phép Agent chia nhỏ mục tiêu phức tạp thành các bước nhỏ hơn, quyết định lựa chọn công cụ phù hợp nhất, tự đánh giá kết quả từ `Observation` của môi trường, và chủ động điều chỉnh chiến lược lập luận ở các bước tiếp theo.

2. **Độ Tin Cậy (Reliability)**:
   ReAct Agent có thể hoạt động *kém hiệu quả hơn* Chatbot truyền thống trong trường hợp các API bên ngoài bị lỗi hoặc độ trễ mạng quá lớn. Do ReAct phụ thuộc hoàn toàn vào chuỗi các bước gọi công cụ tuần tự (mỗi bước yêu cầu một chu kỳ LLM đầy đủ và một yêu cầu mạng đồng bộ), bất kỳ sự cố nghẽn mạng nào cũng sẽ tích lũy độ trễ toàn cục lên rất cao. Nếu mạng bị mất kết nối hoàn toàn và hệ thống không có chế độ dự phòng thông minh, Chatbot thường vẫn có thể trả lời dựa trên tri thức có sẵn, trong khi ReAct Agent sẽ liên tục thử nghiệm gọi công cụ thất bại, gây ra trải nghiệm gián đoạn.

3. **Quan Sát (Observation)**:
   Phản hồi từ môi trường (`Observation`) là các kênh giác quan của Agent. Khi nhận dạng được kết quả phản hồi lỗi từ API hoặc timeout, Agent lập tức điều chỉnh hướng đi của mình. Ví dụ, khi nhận diện được lỗi `Semantic Scholar API returned status code 429`, Agent viết `Thought` thừa nhận việc nghẽn mạng, tự động quyết định chuyển sang `search_arxiv` hoặc chuyển sang tri thức cục bộ để tạo câu trả lời chân thực thay vì cố bịa ra các đường dẫn giả.

---

## IV. Hướng Cải Tiến Trong Tương Lai (5 Điểm)

Để đưa trợ lý khoa học ReAct này lên quy mô hệ thống sản xuất thực tế công nghiệp (production-level), các cải tiến kiến trúc sau cần được thực hiện:

- **Khả Năng Mở Rộng (Scalability)**:
  Tích hợp **Hàng đợi công việc bất đồng bộ (Asynchronous Queue / Task Runner)** như Celery hoặc FastAPI Background Tasks. Thay vì chặn luồng chính của Gradio Web UI trong quá trình tìm kiếm đa bước tốn thời gian, hệ thống có thể thông báo tiến trình từng bước theo thời gian thực và cho phép hàng trăm người dùng truy cập cùng lúc mà không gây nghẽn tài nguyên máy chủ.

- **Tính An Toàn (Safety)**:
  Triển khai mô hình **Supervisor / Auditor** song song. Một mô hình LLM nhỏ hơn, nhanh hơn (hoặc bộ lọc quy tắc nghiêm ngặt) sẽ kiểm duyệt các đối số của công cụ trước khi gọi để ngăn chặn các cuộc tấn công tiêm nhiễm Prompt (Prompt Injection), đồng thời kiểm tra chéo câu trả lời cuối cùng đối chiếu với các `Observation` thực tế để đảm bảo không bị rò rỉ dữ liệu hoặc xuất hiện thông tin sai lệch.

- **Hiệu Năng (Performance)**:
  Thay thế Mock database đơn giản bằng một **Cơ Sở Dữ Liệu Vector (Vector Database)** như Qdrant hoặc Chroma để lập chỉ mục cho kho dữ liệu bài báo mã nguồn mở cục bộ (ví dụ: các bài báo chất lượng cao được tải về trước từ arXiv). Điều này cho phép thực hiện tìm kiếm ngữ nghĩa tương tự (Semantic Search) ngay lập tức khi các API bên ngoài bị sập, mang lại kết quả dự phòng có chất lượng tiệm cận với tìm kiếm trực tuyến thực tế.
