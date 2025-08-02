# **Project Proposal: Triển khai Kiến trúc Tham khảo cho Hệ thống Giám sát và Phân tích Lưu lượng Mạng trên AWS**

## **Một giải pháp Blueprint cho việc nâng cao An ninh và Hiệu suất Vận hành trên nền tảng Cloud**

-----

# **Executive Summary**

Tài liệu này đề xuất một dự án nhằm xây dựng và chứng minh tính hiệu quả của một kiến trúc tham khảo (Reference Architecture) cho hệ thống Giám sát An ninh Mạng (Network Security Monitoring - NSM) trên nền tảng AWS.

  * **Bối cảnh Vấn đề**: Trong các môi trường cloud hiện đại, việc thiếu khả năng quan sát chi tiết các luồng giao tiếp mạng là một thách thức phổ biến. Tình trạng này có thể dẫn đến các điểm mù an ninh, kéo dài thời gian khắc phục sự cố và tăng rủi ro vận hành.
  * **Giải pháp Đề xuất**: Dự án sẽ triển khai một pipeline xử lý dữ liệu, sử dụng các dịch vụ AWS được quản lý hoàn toàn. Giải pháp thu thập **VPC Flow Logs**, sử dụng **Kinesis Data Firehose** để chuyển tiếp dữ liệu đến **Amazon OpenSearch Service** cho việc phân tích thời gian thực và lưu trữ dài hạn trên **Amazon S3**. Một **EC2 Bastion Host** sẽ đảm bảo quyền truy cập quản trị an toàn.
  * **Mục tiêu của Dự án**:
    1.  Xây dựng một Proof of Concept (PoC) hoạt động đầy đủ của kiến trúc đề xuất thông qua phương pháp triển khai thủ công.
    2.  Tạo ra một bộ dashboard mẫu để trực quan hóa dữ liệu mạng.
    3.  Phát triển một bộ tài liệu hướng dẫn triển khai chi tiết từng bước, cho phép người khác có thể tái tạo kiến trúc.
  * **Chi phí và Thời gian**:
      * Dự án được thiết kế để tối ưu chi phí bằng cách tận dụng **AWS Free Tier**. Chi phí vận hành sau khi hết Free Tier cho một môi trường nhỏ được ước tính **dưới $50/tháng**.
      * Thời gian thực hiện dự kiến là **4 tuần**.

Kết quả của dự án là một blueprint hoàn chỉnh, cung cấp một giải pháp hiệu quả và tiết kiệm chi phí cho các tổ chức muốn tăng cường khả năng giám sát và bảo mật cho hạ tầng AWS của mình.

-----

# **1. Problem Statement**

### **Current Situation**

Khi các tổ chức mở rộng hạ tầng trên AWS, kiến trúc mạng trở nên phức tạp hơn với nhiều VPC, subnets và microservices. Trong bối cảnh đó, việc giám sát thủ công thông qua các công cụ cơ bản không còn đủ khả năng đáp ứng.

### **Key Challenges**

Các thách thức kỹ thuật phổ biến mà nhiều tổ chức phải đối mặt bao gồm:

  * **Lack of Network Visibility**: Không có một cái nhìn tập trung và trực quan về các luồng giao tiếp mạng, gây khó khăn trong việc xác định các hành vi bất thường hoặc các điểm nghẽn hiệu năng.
  * **Prolonged Troubleshooting Time**: Khi xảy ra sự cố kết nối, các kỹ sư thường mất nhiều giờ để kiểm tra qua các lớp cấu hình (Security Groups, NACLs, Route Tables) nhằm tìm ra nguyên nhân gốc rễ.
  * **Difficulty in Security Auditing**: Việc thiếu dữ liệu lịch sử về lưu lượng mạng khiến cho các hoạt động điều tra an ninh (forensic analysis) và kiểm toán tuân thủ (compliance audit) trở nên phức tạp và tốn kém.

### **Business Consequences**

Những thách thức kỹ thuật này có thể dẫn đến các hậu quả kinh doanh đáng kể, bao gồm gia tăng rủi ro bị tấn công mạng, giảm năng suất của đội ngũ kỹ thuật và chậm trễ trong việc cung cấp dịch vụ đến người dùng cuối.

-----

# **2. Solution Architecture**

### **Architecture Overview**

Giải pháp được đề xuất là một kiến trúc tham khảo có khả năng mở rộng, linh hoạt và được xây dựng dựa trên các thực tiễn tốt nhất của AWS.

### **AWS Services Used**

  * **Amazon EC2 (Elastic Compute Cloud)**: Sử dụng để khởi chạy một Bastion Host, cung cấp một điểm truy cập an toàn để quản lý các tài nguyên trong mạng riêng tư.
  * **VPC Flow Logs**: Nguồn dữ liệu chính, ghi lại metadata của IP traffic.
  * **Amazon Kinesis Data Firehose**: Dịch vụ streaming được quản lý hoàn toàn, có nhiệm vụ thu thập và phân phối log đến các đích.
  * **Amazon OpenSearch Service**: Cung cấp khả năng tìm kiếm, phân tích và trực quan hóa dữ liệu mạnh mẽ thông qua giao diện OpenSearch Dashboards.
  * **Amazon S3**: Dịch vụ lưu trữ đối tượng, được sử dụng để lưu trữ dữ liệu log thô một cách bền bỉ và chi phí thấp.
  * **Amazon Athena**: Dịch vụ truy vấn serverless, cho phép phân tích dữ liệu trên S3 bằng cú pháp SQL tiêu chuẩn.
  * **AWS IAM**: Quản lý truy cập và phân quyền chi tiết cho các dịch vụ.

### **Component Design**

  * **Data Pipeline**: Thiết kế một pipeline "serverless", giảm thiểu nhu cầu quản lý hạ tầng. Dữ liệu từ VPC Flow Logs được đẩy thẳng đến Kinesis Data Firehose.
  * **Real-time Analytics**: Dữ liệu được đưa vào OpenSearch và lập chỉ mục gần như ngay lập tức, cho phép người dùng thực hiện các truy vấn và xem dashboard cập nhật theo thời gian thực.
  * **Long-term Storage & Ad-hoc Query**: Dữ liệu thô trên S3 được tổ chức theo cấu trúc phân vùng (partitioning) để tối ưu hóa hiệu suất và chi phí khi truy vấn bằng Athena.

### **Security Architecture**

  * **Encryption**: Dữ liệu được mã hóa cả trong quá trình truyền (in-transit) và khi lưu trữ (at-rest).
  * **Network Isolation**: OpenSearch domain được triển khai bên trong VPC, chỉ cho phép truy cập từ các nguồn tin cậy đã được định nghĩa trong Security Group.

### **Scalability Design**

Kiến trúc này có khả năng mở rộng tự nhiên. Các dịch vụ serverless (Kinesis, S3, Athena) tự động điều chỉnh theo khối lượng công việc. Cụm OpenSearch có thể được mở rộng quy mô (scale-up hoặc scale-out) khi cần thiết.

-----

# **3. Technical Implementation**

### **Implementation Phases**

Dự án được cấu trúc thành 4 giai đoạn, mỗi giai đoạn kéo dài một tuần.

  * **Giai đoạn 1**: Thiết lập hạ tầng và bảo mật nền tảng.
  * **Giai đoạn 2**: Triển khai pipeline thu thập và xử lý dữ liệu.
  * **Giai đoạn 3**: Xây dựng dashboard phân tích và kiểm thử.
  * **Giai đoạn 4**: Tối ưu hóa, hoàn thiện tài liệu và đóng gói dự án.

### **Technical Requirements**

  * **Compute**: 1 instance `t3.micro` cho EC2 Bastion Host và 1 instance `t3.small.search` cho OpenSearch (phù hợp cho PoC và thuộc AWS Free Tier).
  * **Storage**: 10 GB EBS cho OpenSearch và 5 GB lưu trữ trên S3 Standard (nằm trong giới hạn Free Tier).
  * **Network**: Một môi trường VPC có sẵn để triển khai và thử nghiệm.

### **Development Approach**

Dự án sẽ áp dụng các phương pháp quản lý linh hoạt, tập trung vào việc tạo ra các sản phẩm bàn giao (deliverables) cụ thể sau mỗi giai đoạn.

### **Testing Strategy**

  * **End-to-End Testing**: Xác minh toàn bộ luồng dữ liệu, từ lúc log được tạo ra cho đến khi nó có thể được truy vấn trên OpenSearch và Athena.
  * **Dashboard Validation**: Kiểm tra tính chính xác và hữu ích của các biểu đồ trên dashboard.

### **Deployment Plan**

  * **Manual Deployment via AWS Management Console**: Toàn bộ hạ tầng sẽ được triển khai thủ công thông qua giao diện đồ họa (GUI) của AWS. Phương pháp này phù hợp cho việc học tập, gỡ lỗi trực quan và tạo ra các tài liệu hướng dẫn chi tiết từng bước. Mỗi bước cấu hình sẽ được chụp ảnh màn hình và ghi chú cẩn thận để đảm bảo người khác có thể làm theo.

-----

# **4. Timeline & Milestones**

### **Project Timeline**

| Hoạt động | Tuần 1 | Tuần 2 | Tuần 3 | Tuần 4 |
| :--- | :---: | :---: | :---: | :---: |
| **1. Foundation Setup** | ██████ | | | |
| **2. Data Pipeline Deployment** | | ██████ | | |
| **3. Analytics & Visualization** | | | ██████ | |
| **4. Documentation & Finalization** | | | | ██████ |

### **Key Milestones**

  * **Kết thúc Tuần 1**: Hạ tầng mạng và bảo mật được triển khai thành công.
  * **Kết thúc Tuần 2**: Luồng dữ liệu hoạt động, log được ghi nhận tại S3 và OpenSearch.
  * **Kết thúc Tuần 3**: Dashboard phiên bản 1.0 hoàn thành.
  * **Kết thúc Tuần 4**: Hoàn thành dự án, bàn giao bộ tài liệu hướng dẫn triển khai chi tiết.

### **Dependencies**

  * Yêu cầu một tài khoản AWS để triển khai các tài nguyên.

### **Resource Allocation**

  * Đây là một dự án cá nhân, do một người thực hiện.

-----

# **5. Budget Estimation**

### **Infrastructure Costs (AWS)**

Bảng dưới đây ước tính chi phí hàng tháng cho một môi trường PoC nhỏ, sau khi đã sử dụng hết các quyền lợi từ AWS Free Tier.
| Dịch vụ | Chi phí sau Free Tier (ước tính) |
| :--- | :--- |
| **Amazon OpenSearch Service** (`t3.small`) | \~$25/tháng |
| **Amazon EC2** (`t3.micro`) | \~$5/tháng |
| **Amazon S3** (dưới 10 GB) | \<$1/tháng |
| **Kinesis Data Firehose** (dưới 10 GB) | \~$5/tháng |
| **VPC Flow Logs** & **Athena** | \~$10/tháng |
| **Tổng cộng** | **\~$46 / tháng** |

### **ROI Analysis**

Đối với một dự án mang tính chất xây dựng kiến trúc tham khảo, ROI không được đo lường trực tiếp bằng tài chính mà qua các giá trị sau:

  * **Giá trị Kỹ thuật**: Cung cấp một giải pháp đã được kiểm chứng (proven solution) giúp giảm thời gian và rủi ro khi triển khai trong thực tế.
  * **Giá trị Vận hành**: Các tổ chức áp dụng kiến trúc này có thể tiết kiệm hàng chục giờ làm việc mỗi tháng cho đội ngũ kỹ thuật.
  * **Giá trị Tri thức**: Bộ tài liệu hướng dẫn chi tiết là một tài sản tri thức có giá trị, có thể được sử dụng để đào tạo và phát triển.

-----

# **6. Risk Assessment**

### **Risk Matrix**

| Rủi ro | Mức độ Ảnh hưởng | Khả năng xảy ra | Chiến lược giảm thiểu |
| :--- | :--- | :--- | :--- |
| **Kỹ thuật**: Gặp phải vấn đề tích hợp phức tạp | Trung bình | Trung bình | Phân bổ thêm thời gian cho giai đoạn nghiên cứu và thử nghiệm. |
| **Phạm vi**: Yêu cầu của dự án vượt quá thời gian | Cao | Thấp | Bám sát các mục tiêu cốt lõi đã định nghĩa (MVP). Các tính năng nâng cao sẽ được ghi nhận như đề xuất cho tương lai. |
| **Chi phí**: Phát sinh chi phí ngoài dự kiến | Thấp | Thấp | Sử dụng AWS Budgets để thiết lập cảnh báo chi phí. Thường xuyên kiểm tra và dọn dẹp tài nguyên sau khi thử nghiệm. |

-----

# **7. Expected Outcomes**

### **Success Metrics**

  * **Deliverables**:
      * Một hệ thống PoC hoạt động đầy đủ.
      * Một bộ tài liệu hướng dẫn triển khai từng bước chi tiết, có hình ảnh minh họa.
      * Một bài thuyết trình tổng kết dự án.
  * **Performance**:
      * Thời gian từ khi log được tạo ra đến khi xuất hiện trên dashboard dưới 2 phút.

### **Business Benefits**

  * **Tăng tốc triển khai**: Các tổ chức có thể áp dụng blueprint này để nhanh chóng xây dựng một hệ thống giám sát hiệu quả.
  * **Giảm rủi ro**: Cung cấp một giải pháp đã được kiểm chứng, tuân thủ các thực tiễn tốt nhất về bảo mật và vận hành.

### **Long-term Value**

Dự án này tạo ra một kiến trúc tham khảo có giá trị, có thể được cộng đồng và các tổ chức khác nhau áp dụng. Nó thể hiện một cách tiếp cận hiện đại để giải quyết một bài toán kinh điển trong lĩnh vực an ninh và vận hành đám mây, đồng thời thúc đẩy văn hóa ra quyết định dựa trên dữ liệu.