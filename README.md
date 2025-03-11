# list
from flask import Flask, render_template, request, jsonify
import sqlite3
import paho.mqtt.client as mqtt
import datetime
import matplotlib
matplotlib.use("Agg")  # Fix Tkinter error by using Agg backend
import matplotlib.pyplot as plt
import io
import base64
import matplotlib.dates as mdates

app = Flask(__name__)
DATABASE = "new_status.db"

# MQTT Configuration
MQTT_BROKER = "test.mosquitto.org"
MQTT_PORT = 1883
MQTT_TOPIC = "water_mgmt/mode"

mqtt_client = mqtt.Client()

# Handle MQTT Connection
def on_connect(client, userdata, flags, rc):
    print(f"Connected to MQTT Broker with result code {rc}")
    client.subscribe(MQTT_TOPIC)

# Handle Received MQTT Messages
def on_message(client, userdata, msg):
    status = msg.payload.decode().strip()
    print(f"Received MQTT Message: {status}")
    
    if status in ["ON", "OFF", "AUTO"]:
        insert_status(status)

mqtt_client.on_connect = on_connect
mqtt_client.on_message = on_message
mqtt_client.connect(MQTT_BROKER, MQTT_PORT, 60)
mqtt_client.loop_start()  # Fix: No need for threading.Thread()

# Initialize SQLite Database
def init_db():
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS status_history (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                status TEXT NOT NULL,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        """)
        conn.commit()

# Insert Status into Database
def insert_status(status):
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute("INSERT INTO status_history (status) VALUES (?)", (status,))
        conn.commit()

# Get Latest Status
def get_latest_status():
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT status FROM status_history ORDER BY id DESC LIMIT 1")
        result = cursor.fetchone()
    return result[0] if result else "OFF"

# Generate Status Graph
def generate_status_graph():
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT timestamp, status FROM status_history ORDER BY timestamp ASC")
        data = cursor.fetchall()

    if not data:
        return None

    timestamps, statuses = zip(*data)
    timestamps = [datetime.datetime.strptime(ts, "%Y-%m-%d %H:%M:%S") for ts in timestamps]

    status_mapping = {"OFF": 0, "ON": 1, "AUTO": 2}
    numeric_statuses = [status_mapping[status] for status in statuses]

    plt.figure(figsize=(8, 4))
    plt.plot(timestamps, numeric_statuses, marker='o', linestyle='-', color='b')
    plt.xlabel("Timestamp")
    plt.ylabel("Status (0 = OFF, 1 = ON, 2 = AUTO)")
    plt.title("Status History Over Time")
    plt.xticks(rotation=45)
    plt.yticks([0, 1, 2], ["OFF", "ON", "AUTO"])
    plt.gca().xaxis.set_major_formatter(mdates.DateFormatter("%Y-%m-%d %H:%M"))
    plt.grid()

    img = io.BytesIO()
    plt.savefig(img, format='png', bbox_inches="tight")
    img.seek(0)
    graph_url = base64.b64encode(img.getvalue()).decode()
    plt.close()
    return f"data:image/png;base64,{graph_url}"

# API: Set Status via AJAX and MQTT
@app.route('/set_status_ajax', methods=['POST'])
def set_status_ajax():
    data = request.get_json()
    if not data or "status" not in data:
        return jsonify({"success": False, "error": "Invalid data received"})

    new_status = data["status"]
    if new_status in ["ON", "OFF", "AUTO"]:
        insert_status(new_status)
        mqtt_client.publish(MQTT_TOPIC, new_status)  # Publish to MQTT
        return jsonify({"success": True, "status": new_status})

    return jsonify({"success": False, "error": "Invalid status value"})

# API: Get Status Table
@app.route('/get_table', methods=['GET'])
def get_table():
    with sqlite3.connect(DATABASE) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT id, status, timestamp FROM status_history ORDER BY id DESC")
        status_history = cursor.fetchall()
    return jsonify({"table_data": status_history})

# API: Get Graph Data
@app.route('/get_graph', methods=['GET'])
def get_graph():
    graph_img = generate_status_graph()
    if graph_img:
        return jsonify({"graph": graph_img})
    return jsonify({"error": "No data available"})

# API: Get Latest Status
@app.route('/get_latest_status', methods=['GET'])
def get_latest_status_route():
    return jsonify({"status": get_latest_status()})

# Home Page
@app.route('/index')
def index():
    return render_template('index1.html', status=get_latest_status(), title="Smart Water Management System")

@app.route('/')
def home():
    return render_template('home.html', title="Smart Water Management System")

if __name__ == '__main__':
    init_db()
    app.run(debug=True, use_reloader=False, threaded=False)  # Fix threading issues

