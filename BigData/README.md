## Setup

### Web-Application

#### 1. Cấu trúc thư mục

```
project-folder/
├── app.py
└── templates/
    └── index.html
```

#### `app.py`

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