# Setup

## Web-Application

### 1. Cấu trúc thư mục

```
project-folder/
├── app.py
└── templates/
    └── index.html
```

### File `app.py`

```python
from flask import Flask, render_template, request, jsonify
from flask_socketio import SocketIO, emit
from kafka import KafkaProducer
from flask_cors import CORS
import json

app = Flask(__name__)
CORS(app)
socketio = SocketIO(app, cors_allowed_origins="*")

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/realtime')
def realtime():
    return render_template('realtime.html')

@app.route('/log-mouse', methods=['POST'])
def log_mouse():
    data = request.get_json()
    try:
        for point in data:
            producer.send('mouse-data', point)
        return 'Logged to Kafka successfully', 200
    except Exception as e:
        return f'Error sending to Kafka: {str(e)}', 500

@app.route('/stream', methods=['POST'])
def stream_mouse_data():
    data = request.get_json()
    print(f"[LOG] Mouse Data: {data}")
    socketio.emit('mouse_data', data)
    return jsonify({"status": "ok"}), 200

if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=80)
```

### 2. Cài đặt môi trường trên Ubuntu

#### 2.1. Cài Python + pip + virtualenv (nếu chưa có)

```
sudo apt update
sudo apt install python3 python3-pip python3-venv -y
```

Cài thêm thư viện cần thiết nếu chưa có:

```
pip install kafka-python
pip install flask_socketio
```

#### 2.2. Tạo project folder và môi trường ảo

```
mkdir ~/project-folder
cd ~/project-folder
python3 -m venv venv
source venv/bin/activate
```

#### 2.3. Copy mã vào server

Bạn có thể dùng `scp` hoặc clone từ Git, hoặc tạo file trực tiếp:

```
nano app.py
mkdir templates
nano templates/index.html
```

#### 2.4. Cài Flask & Gunicorn

```
pip install flask gunicorn
```

### 3. Chạy thử với Gunicorn

```
gunicorn -w 4 -b 0.0.0.0:8000 app:app
```
- `-w 4`: 4 worker (tùy server)

- `-b 0.0.0.0:8000`: lắng nghe toàn bộ IP ở port 8000

Test thử truy cập: `http://your-server-ip:8000`

Nếu chạy `gunicorn` bị lỗi có thể thử lại:

```
sudo ~/project-folder/venv/bin/gunicorn -w 4 -b 0.0.0.0:80 app:app
```

Để chạy WebSocket ổn định với Flask + Gunicorn, bạn cần dùng một trong các worker hỗ trợ async, phổ biến nhất là: `eventlet` hoặc `gevent`

#### Cài đặt

```
pip install eventlet
```

Thêm -k eventlet để chỉ định dùng eventlet worker:

```
sudo ~/flask/project-folder/venv/bin/gunicorn -w 1 -k eventlet -b 0.0.0.0:80 realtime_server:app
```

## Cái đặt Kafka trên Ubuntu sử dụng Docker

### 1. Tạo file `docker-compose.yml`

Sử dụng `images` mới nhất của Kafka là `kafka:4.0.0`

```
version: '3.8'
services:
  kafka:
    image: apache/kafka:4.0.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_BROKER_ID: 1
      KAFKA_PROCESS_ROLES: controller,broker
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
```

### 2. Tải và cài đặt Docker:

```
sudo docker compose up -d
```

## Spark

Chạy file `network_mouse_tracking.py` với câu lệnh sau:

```
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.5 network_mouse_tracking.py localhost 9999
```

- Lưu ý cần kiểm tra đúng version của `spark-sql-kafka-0-10_2.12:3.5.5`


Tài file từ server về máy local:

```
scp root@server-ip:/home/user/path/file-name C:\Users\username\Downloads
```
