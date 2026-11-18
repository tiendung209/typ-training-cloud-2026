MONITORING VÀ ALERTING
Mục đích của việc giám sát trong hệ thống cloud-native

Một hệ thống cloud-native gồm nhiều microservice chạy trong container. Thì pod sẽ tự tạo ra và xóa đi liên tục không ngừng nghỉ.IP sẽ đổi liên tục, và kubernetes tự scale và tự restart . Không có người nào ngồi canh từng máy xem lúc nào restart lúc nào scale,.. nên cần một hệ thống tự thu thập số liệu và báo khi có chuyện xảy ra

giám sát sẽ có 3 câu hỏi
Hệ thống có khỏe không
Vì sao service hỏng,
có còn đủ tài nguyên để chạy tiếp không
observability có 3 trụ cột chính là metric log và traces một khung hay dùng để chọn metric càn theo dõi là 4 golden signature latency traffic và errors, saturation

2.CÁC LOẠI METRIC TRONG K8S
Có thể nghĩ theo 3 tầng, từ dưới lên
node metric :sức khỏe của máy chủ(cpu, ram, disk, network) thương do node-exporter cung cấp
Pod/cont metricL mỗi cont đang dùng bao nhiêu cpu/ram, restart bao nhiêu lần,... Trạng thái các đối tượng k8s đến từ kube-statre-metrics(pending, runing, deployment)
Application metrics : số liệu đo chính ứng dụng tự do như request thời gian phản hồi, số lỗi . Ứng dụng dùng thư viện client của prometheus và mở endpoint/metric
Hai tầng đầu sẽ cho biết máy có còn chịu được nữa hay không, còn tầng ứng dụng cho biết người dùng có đang bị ảnh hưởng bởi hệ thống không. Tầng ứng dụng là tầng mà hạ tầng không thể biết được



3.PROMETHEUS
kiến trúc
prometheus sẽ làm 3 việc thu thập lưu trũ và truy vấn. Xung quanh có exporter (nguồn metrics), service discovery*tự tìm target), alertmanager(xử lí cảnh báo) và grafana (hiện thị thông số)
Cơ chế pull/scraping
Prometheus sẽ chủ động gọi HTTP đến metrics của từng target theo chu kì thương 15-20 s rồi kết quả được trả về một trang text ví dụ như
http_requests_total{method="GET", status="200"} 10234
có một lí do chọn pull thay vì để app push là vì 
Target mà chết thì scape sẽ error ngay, nên phát hiện ngay
Ứng dụng không cần biết địa chỉ của hệ thống giám sát
Với k8s, prometheus sẽ dùng service discovery để tự thấy pod mới, nên không phải cấu hình tay
Dễ kiểm tra thủ công /mở metrics bằng trình duyệt là tìm thấy rồi

Time-series DB và PromQL
Một chuỗi dữ liệu được định danh bằng tên metric+các label:
http_requests_total{method="GET", status="200"}  →  (thời điểm, giá trị), (thời điểm, giá trị), ...
Label rất mạnh vì cho phép lọc và gom nhóm theo pod, namespace, status code cần có 4 kiểu metric
-counter: chỉ tẳng tổng số request
-gauge lên xuống ram 
-histogramz: phân bố giá trị theo các khoảng latency
-summary tương tự như histogram nhưng tính range ngay tại ứng dụng
PromQL là ngôn ngữ truy vấn của Prometheus có 3 query cần nhớ 
# Số request mỗi giây trong 5 phút gần nhất
rate(http_requests_total[5m])

# CPU dùng theo từng pod
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))

# Độ trễ phân vị 95
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

4.Grafana Dashboards
Grafana sẽ không lưu trữ tệp, nó sẽ chỉ kết nối đến data soure (prometheus) gửi query promql để rồi vẽ bieur đồ mỗi panel trên dashboard thực chất là một query điểm mạnh là dashboard có biến (namespace, pod) và có thể import dashboard dựng sẵn từ cộng đồng ví dụ node exporter full. Prometheus có giao dinej riêng nhưng chỉ phù hợp để thử query grafana mới là nới để xem thông tin cụ thể hàng ngày hơn


5.Cảnh báo và alermanager
prometheus sẽ quyết định khi nào có cảnh báo bằng alert rule
Ví dụ 
- alert: HighCPU
  expr: sum by (pod) (rate(container_cpu_usage_seconds_total[5m])) > 0.9
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.pod }} dùng CPU quá cao"
thì ở đây từ khóa for 5m nghĩa là điều kiện phải đúng liên tục 5 phút mới báo để tránh nhầm do dao động một thời điểm quá ngắn đôi khi biến số xảy ra trong 1,2 giây là chuyện bình thường. Alert sẽ đi qua 2 trạng thái inactive--> pending __ firing

Alertmanage sẽ quyết định gửi thế nào nhân alert đang firing rồi 
Gom nhóm lại: ví dụ 50 pod lỗi gom thành 1 gửi thông báo
Chống trùng lặp lỗi
định tuyến lỗi nghiêm trọng gửi pageduy lỗi nhẹ sẽ gửi email
Silence tạm ngắt khi bảo trì hoặc chặn cảnh báo phụ khi đã có cảnh báo gốc

CƠ CHẾ KIỂM TA SỨC KHỎE ỨNG DỤNG
Bản chất la cơ chế k8s tự kiểm tra ứng dụng khác với prometheu là con người quan sát
Liveness probe sẽ hỏi ứng dụng còn sống không, nếu chết rôi kublet restart container dùng để cứu trường hớp ứng dụng và treo ma process báo vẫn còn
Readiness probe hỏi ứng dụng đã sẵn sàng nhận request chưa nếu thất bại pod không bị restart chỉ bị gơ khoi sevice dể khong nhận traffic dùng khi app đang khởi động đang nạp cache hoặc bị quá tải