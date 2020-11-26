## ETL Project Structure

The basic project structure is as follows:

```bash
root/
 |-- configs/
 |   |-- etl_config.json    (Chứa các tham số cấu hình dạng json)
 |-- dependencies/          (Module bổ sung )
 |   |-- logging.py         (Hiển thị các log)
 |   |-- spark.py           (Khởi tạo spark session)
 |-- jobs/
 |   |-- etl_job.py         (Chương trình chính cần submit lên cluster manager) 
 |-- tests/                 
 |   |-- test_data/
 |   |-- | -- employees/
 |   |-- | -- employees_report/
 |   |-- test_etl_job.py    (Chứa module kiểm tra dữ liệu)
 |   build_dependencies.sh  (Đóng gói dependencies thành packages.zip để gửi theo chương trình)
 |   packages.zip
 |   Pipfile                (Các dependencies cho môi trường pipenv để project hoạt động)
 |   Pipfile.lock
```

## Passing Configuration Parameters to the ETL Job

Thông thường có thể truyền đối số vào ngay sau spark-submit. Tuy nhiên thì làm vậy rất phức tạp và khó debug vì vấn đề về quyền truy cập

Một giải pháp hiệu quả hơn nhiều là gửi cho Spark một tệp riêng - ví dụ: bằng cách sử dụng cờ `--files configs/etl_config.json` với` spark-submit` - chứa cấu hình ở định dạng JSON. Việc kiểm tra mã từ bên trong driver tương tác Python cũng được đơn giản hóa rất nhiều, vì tất cả những gì người ta phải làm để truy cập các thông số cấu hình để kiểm tra, là sao chép và dán nội dung của tệp - ví dụ:

```python
import json

config = json.loads("""{"field": "value"}""")
```

## Packaging ETL Job Dependencies

Trong dự án này, các chức năng có thể được sử dụng trên các công việc ETL khác nhau được lưu giữ trong một mô-đun có tên là `dependencies` và được tham chiếu trong các mô-đun công việc cụ thể bằng cách sử dụng, ví dụ:

```python
from dependencies.spark import start_spark
```

Gói này, cùng với bất kỳ dependency bổ sung nào được tham chiếu trong nó, phải được sao chép vào mỗi nút Spark cho tất cả các công việc sử dụng `dependencies` để chạy. Điều này có thể đạt được bằng một trong số các cách:

1. gửi tất cả các phụ thuộc dưới dạng kho lưu trữ `zip` cùng với công việc, sử dụng` --py-files` với Spark submit;
2. chính thức đóng gói và tải các `dependencies` lên một nơi nào đó như kho lưu trữ` PyPI` (hoặc phiên bản riêng tư) và sau đó chạy `pip3 install dependencies` trên mỗi nút; hoặc là,
3. kết hợp giữa việc sao chép thủ công các mô-đun mới (ví dụ: `dependencies`) vào đường dẫn Python của mỗi nút và sử dụng` pip3 install` cho các phụ thuộc bổ sung

## Running the ETL job

Giả sử rằng biến môi trường `$SPARK_HOME` trỏ đến thư mục cài đặt Spark cục bộ của bạn, thì công việc ETL có thể được chạy từ thư mục gốc của dự án bằng cách sử dụng lệnh sau từ terminal,

```bash
$SPARK_HOME/bin/spark-submit \
--master local[*] \
--packages 'com.somesparkjar.dependency:1.0.0' \
--py-files packages.zip \
--files configs/etl_config.json \
jobs/etl_job.py
```

Briefly, the options supplied serve the following purposes:

- `--master local [*]` - địa chỉ của cụm Spark để bắt đầu công việc. Nếu bạn có một cụm Spark đang hoạt động (ở chế độ một người thực thi cục bộ hoặc một cái gì đó lớn hơn trong đám mây) và muốn gửi công việc đến đó, thì hãy sửa đổi điều này bằng Spark IP thích hợp - ví dụ: `spark: // the-cluster-ip-address: 7077`;

- `--packages 'com.somesparkjar.dependency: 1.0.0, ...'`'- Maven tọa độ cho bất kỳ phụ thuộc JAR nào theo yêu cầu của công việc (ví dụ: trình điều khiển JDBC để kết nối với cơ sở dữ liệu quan hệ);

- `--files configs/etl_config.json` - đường dẫn (tùy chọn) đến bất kỳ tệp cấu hình nào có thể được yêu cầu bởi công việc ETL;

- `--py-files packages.zip` - lưu trữ chứa các phụ thuộc Python (mô-đun) được tham chiếu bởi công việc; và,

- `jobs/etl_job.py` - tệp mô-đun Python có chứa công việc ETL để thực thi.
