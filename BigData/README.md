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

Cài thêm thư viên nếu chưa có:

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
sudo ~/flask/project-folder/venv/bin/gunicorn -w 4 -b 0.0.0.0:80 app:app
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
