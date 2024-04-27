# Network
#HTTP_SERVER
import http.server
import socketserver
import os
import base64

# Directory where dynamic content scripts are stored
SCRIPTS_DIR = "scripts"

# HTTP Request Handler
class RequestHandler(http.server.SimpleHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length)
        self.send_response(200)
        self.end_headers()
        self.wfile.write(post_data)

    def do_AUTHHEAD(self):
        self.send_response(401)
        self.send_header('WWW-Authenticate', 'Basic realm=\"Test\"')
        self.send_header('Content-type', 'text/html')
        self.end_headers()

    def do_GET(self):
        if self.headers.get('Authorization') is None:
            self.do_AUTHHEAD()
            self.wfile.write(bytes("Authentication required", 'utf-8'))
        elif self.authenticate(self.headers.get('Authorization')):
            if self.path.endswith(".py"):
                # Handle dynamic content
                self.handle_dynamic_content()
            else:
                # Serve static content
                http.server.SimpleHTTPRequestHandler.do_GET(self)
        else:
            self.do_AUTHHEAD()
            self.wfile.write(bytes("Authentication failed", 'utf-8'))

    def authenticate(self, auth_header):
        try:
            encoded_credentials = auth_header.split(' ')[1]
            decoded_credentials = base64.b64decode(encoded_credentials).decode('utf-8')
            username, password = decoded_credentials.split(':')
            return username == "user" and password == "pass"
        except:
            return False

    def handle_dynamic_content(self):
        script_name = self.path[1:]  # Remove leading slash
        script_path = os.path.join(SCRIPTS_DIR, script_name)
        
        if os.path.exists(script_path):
            try:
                with open(script_path, 'rb') as file:
                    script_content = file.read()
                exec(script_content, globals(), locals())
                response_content = locals().get("content", "Dynamic content generation failed.")
            except Exception as e:
                response_content = f"Error: {str(e)}"
        else:
            response_content = "404 Not Found"
        
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(bytes(response_content, 'utf-8'))

# Start HTTP server
def start_server():
    with socketserver.TCPServer(("", 9000), RequestHandler) as httpd:
        print("HTTP server started at port 9000")
        httpd.serve_forever()

# Start the server
if _name_ == "_main_":
    start_server()
----------------------------
#HTTP_CLIENT
import sys
from PyQt5.QtWidgets import QApplication, QMainWindow, QLabel, QLineEdit, QPushButton, QComboBox, QTextEdit
import requests

class PyBrowserWindow(QMainWindow):
    def _init_(self):
        super()._init_()
        self.setWindowTitle("PyBrowser")
        self.setGeometry(100, 100, 600, 400)

        # URL Input
        self.url_label = QLabel(self)
        self.url_label.setText("URL:")
        self.url_label.move(20, 20)
        self.url_entry = QLineEdit(self)
        self.url_entry.setGeometry(100, 20, 400, 30)

        # Method Combobox
        self.method_label = QLabel(self)
        self.method_label.setText("Method:")
        self.method_label.move(20, 60)
        self.method_combobox = QComboBox(self)
        self.method_combobox.setGeometry(100, 60, 150, 30)
        self.method_combobox.addItems(["GET", "POST"])

        # Headers Input
        self.headers_label = QLabel(self)
        self.headers_label.setText("Headers:")
        self.headers_label.move(20, 100)
        self.headers_text = QTextEdit(self)
        self.headers_text.setGeometry(100, 100, 400, 80)

        # Request Body Input
        self.body_label = QLabel(self)
        self.body_label.setText("Request Body:")
        self.body_label.move(20, 200)
        self.body_text = QTextEdit(self)
        self.body_text.setGeometry(100, 200, 400, 100)

        # Send Button
        self.send_button = QPushButton(self)
        self.send_button.setText("Send Request")
        self.send_button.setGeometry(250, 320, 150, 30)
        self.send_button.clicked.connect(self.send_request)

        # Response Label
        self.response_label = QLabel(self)
        self.response_label.setText("Response:")
        self.response_label.move(20, 360)

        # Response Text
        self.response_text = QTextEdit(self)
        self.response_text.setGeometry(100, 360, 400, 100)

    def send_request(self):
        url = self.url_entry.text()
        method = self.method_combobox.currentText()
        headers = self.parse_headers(self.headers_text.toPlainText())
        body = self.body_text.toPlainText()

        try:
            if method == "GET":
                response = requests.get(url, headers=headers)
            elif method == "POST":
                response = requests.post(url, headers=headers, data=body)
            
            self.response_text.setText(response.text)
        except Exception as e:
            self.response_text.setText(f"Error: {str(e)}")

    def parse_headers(self, headers_text):
        headers = {}
        if headers_text.strip():
            lines = headers_text.split('\n')
            for line in lines:
                key, value = line.split(':', 1)
                headers[key.strip()] = value.strip()
        return headers

if _name_ == "_main_":
    app = QApplication(sys.argv)
    window = PyBrowserWindow()
    window.show()
    sys.exit(app.exec_())
--------------------------------------
#HTML_FILE
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Static Content</title>
</head>
<body>
    <h1>This is a static HTML page</h1>
    <p>Hello User</p>
    <!-- Adding the image -->
    <img src="data:image/png;base64,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" alt="PPU Picture">
</body>
</html>
