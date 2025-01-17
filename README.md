# Monitor-and-Limit-Wi-Fi-Usage-via-Flask-Server
#Python-based local web server to monitor and limit Wi-Fi usage. This version logs data usage per student (based on their IP) and limits access if their usage exceeds the predefined threshold.
from flask import Flask, jsonify, request
import psutil
import subprocess
import time
import threading

# Flask app for monitoring
app = Flask(__name__)

# Configuration
USAGE_LIMIT_MB = 500  # Example: Limit in MB
CHECK_INTERVAL = 60   # Check interval in seconds
students = {
    "192.168.1.101": {"name": "Student1", "usage": 0, "blocked": False},
    "192.168.1.102": {"name": "Student2", "usage": 0, "blocked": False}
}

# Monitor Wi-Fi usage
def monitor_wifi_usage():
    net_io_start = psutil.net_io_counters(pernic=True).get('wlan0', None)
    if not net_io_start:
        print("Wi-Fi adapter not found!")
        return

    start_bytes = net_io_start.bytes_sent + net_io_start.bytes_recv

    while True:
        time.sleep(CHECK_INTERVAL)
        net_io_current = psutil.net_io_counters(pernic=True).get('wlan0', None)
        if not net_io_current:
            continue

        current_bytes = net_io_current.bytes_sent + net_io_current.bytes_recv
        usage_mb = (current_bytes - start_bytes) / (1024 * 1024)  # Convert to MB

        for ip, data in students.items():
            if not data['blocked']:
                students[ip]["usage"] = usage_mb
                if students[ip]["usage"] > USAGE_LIMIT_MB:
                    block_ip(ip)
                    students[ip]["blocked"] = True

# Function to block IP
def block_ip(ip_address):
    try:
        subprocess.run(["iptables", "-A", "OUTPUT", "-s", ip_address, "-j", "DROP"], check=True)
        print(f"Blocked internet access for IP: {ip_address}")
    except subprocess.CalledProcessError as e:
        print(f"Failed to block IP {ip_address}: {e}")

# Function to unblock IP
def unblock_ip(ip_address):
    try:
        subprocess.run(["iptables", "-D", "OUTPUT", "-s", ip_address, "-j", "DROP"], check=True)
        print(f"Unblocked internet access for IP: {ip_address}")
        students[ip_address]["blocked"] = False
        students[ip_address]["usage"] = 0  # Reset usage
    except subprocess.CalledProcessError as e:
        print(f"Failed to unblock IP {ip_address}: {e}")

# API to view student data
@app.route("/students", methods=["GET"])
def get_students():
    return jsonify(students)

# API to unblock a student
@app.route("/unblock", methods=["POST"])
def unblock_student():
    data = request.json
    ip_address = data.get("ip")
    if ip_address in students:
        unblock_ip(ip_address)
        return jsonify({"message": f"IP {ip_address} unblocked successfully."}), 200
    else:
        return jsonify({"error": "Invalid IP address"}), 400

# Run the Flask app in a thread
def start_flask_app():
    app.run(host="0.0.0.0", port=5000)

# Start the monitoring and Flask server
if __name__ == "__main__":
    monitoring_thread = threading.Thread(target=monitor_wifi_usage, daemon=True)
    monitoring_thread.start()

    start_flask_app()
