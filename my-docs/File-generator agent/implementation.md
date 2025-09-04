# Implementation Blueprint: Dify File-Generation Microservice

This document provides the complete source code and configuration required to build the `file-generator` microservice. This service acts as a bridge between AnythingLLM and a Dify agent, enabling file generation (PDF, DOCX, XLSX) and chat history saving.

---

## 1. Project Structure

The microservice should be created in a new directory, completely separate from the AnythingLLM project.

```
file-generator/
├── static/
│   └── generated_files/
│       └── .gitkeep
├── app.py
├── Dockerfile
└── requirements.txt
```

---

## 2. `requirements.txt`

Create a `requirements.txt` file with the following Python dependencies:

```text
Flask==2.2.2
requests==2.28.1
fpdf==1.7.2
python-docx==0.8.11
openpyxl==3.0.10
```

---

## 3. `Dockerfile`

Create a `Dockerfile` to containerize the Flask application.

```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Copy the dependencies file to the working directory
COPY requirements.txt .

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Copy the content of the local src directory to the working directory
COPY . .

# Make port 5001 available to the world outside this container
EXPOSE 5001

# Define environment variable
ENV FLASK_APP=app.py

# Run the command to start the server
CMD ["flask", "run", "--host=0.0.0.0", "--port=5001"]
```

---

## 4. `app.py` (Full Source Code)

This is the main application file. Create an `app.py` file and paste the following code into it.

```python
import os
import uuid
from flask import Flask, request, jsonify, send_from_directory
import requests
from fpdf import FPDF
from docx import Document
from openpyxl import Workbook

app = Flask(__name__)

# Configuration
GENERATED_FILES_DIR = os.path.join('static', 'generated_files')
if not os.path.exists(GENERATED_FILES_DIR):
    os.makedirs(GENERATED_FILES_DIR)

# --- Helper Functions for File Generation ---

def create_pdf(content, filename):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    # Add content to the PDF, handling multi-line text
    for line in content.split('\n'):
        pdf.multi_cell(0, 10, txt=line)
    
    filepath = os.path.join(GENERATED_FILES_DIR, filename)
    pdf.output(filepath)
    return filepath

def create_docx(content, filename):
    document = Document()
    document.add_paragraph(content)
    filepath = os.path.join(GENERATED_FILES_DIR, filename)
    document.save(filepath)
    return filepath

def create_xlsx(content, filename):
    workbook = Workbook()
    sheet = workbook.active
    # Split content by lines and then by a delimiter (e.g., comma) for cells
    for r_idx, row_data in enumerate(content.split('\n'), 1):
        for c_idx, cell_value in enumerate(row_data.split(','), 1):
            sheet.cell(row=r_idx, column=c_idx, value=cell_value.strip())

    filepath = os.path.join(GENERATED_FILES_DIR, filename)
    workbook.save(filepath)
    return filepath

# --- API Endpoints ---

@app.route('/anythingllm-manifest.json', methods=['GET'])
def manifest():
    return jsonify({
        "version": "1.0.2",
        "name": "Dify File Generator",
        "description": "An agent that can generate files (PDF, DOCX, XLSX), save chat history, or save the last response using a Dify agent.",
        "type": "agent",
        "entrypoint": f"{request.host_url}invoke",
        "schemas": [
            {
                "type": "agent",
                "name": "dify-file-generator",
                "provider": "dify",
                "description": "Use this agent to create files from content, save the full conversation, or save the last response.",
                "settings": [
                    {
                        "name": "Dify API Endpoint",
                        "type": "text",
                        "description": "Your Dify application's API endpoint (e.g., https://api.dify.ai/v1)",
                        "value": "",
                        "required": True
                    },
                    {
                        "name": "Dify API Secret Key",
                        "type": "secret",
                        "description": "Your Dify application's secret key.",
                        "value": "",
                        "required": True
                    }
                ],
                "examples": [
                    "Create a PDF named 'report.pdf' with the content 'This is a test'.",
                    "Save this conversation as a DOCX file named 'chat_log.docx'.",
                    "Save the last response as a file named 'summary.pdf'."
                ]
            }
        ]
    })

@app.route('/generate/<file_type>', methods=['POST'])
def generate_file(file_type):
    data = request.get_json()
    content = data.get('content', '')
    filename = data.get('filename')

    if not filename:
        return jsonify({"error": "Filename is required"}), 400

    try:
        if file_type == 'pdf':
            filepath = create_pdf(content, filename)
        elif file_type == 'docx':
            filepath = create_docx(content, filename)
        elif file_type == 'xlsx':
            filepath = create_xlsx(content, filename)
        else:
            return jsonify({"error": "Unsupported file type"}), 400
        
        file_url = f"{request.host_url}{filepath}"
        return jsonify({"url": file_url})
    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/invoke', methods=['POST'])
def invoke_dify_agent():
    data = request.get_json()
    user_prompt = data.get('prompt', '')
    conversation_history = data.get('history', [])
    settings = data.get('settings', {})

    dify_api_endpoint = settings.get('Dify API Endpoint')
    dify_api_key = settings.get('Dify API Secret Key')

    if not dify_api_endpoint or not dify_api_key:
        return jsonify({"text": "Dify API credentials are not configured."})

    # --- Process History for Dify --- #
    # Find the last assistant message
    last_assistant_message = ""
    for message in reversed(conversation_history):
        if message.get('role') == 'assistant':
            last_assistant_message = message.get('content', '')
            break

    # Format the full conversation history into a single string
    formatted_chat_history = ""
    for message in conversation_history:
        role = "User" if message.get('role') == 'user' else "Assistant"
        content = message.get('content', '')
        formatted_chat_history += f"{role}: {content}\n\n"
    # --- End History Processing --- #

    headers = {
        'Authorization': f'Bearer {dify_api_key}',
        'Content-Type': 'application/json'
    }

    payload = {
        'inputs': {
            'last_assistant_message': last_assistant_message,
            'formatted_chat_history': formatted_chat_history.strip()
        },
        'query': user_prompt,
        'response_mode': 'streaming',
        'user': f'anythingllm-user-{uuid.uuid4()}',
        'conversation_id': data.get('conversation_id', ''),
        'files': []
    }

    try:
        response = requests.post(
            f"{dify_api_endpoint}/chat-messages",
            headers=headers,
            json=payload,
            stream=True
        )
        response.raise_for_status()

        # Process streaming response
        final_answer = ""
        for line in response.iter_lines():
            if line:
                decoded_line = line.decode('utf-8')
                if decoded_line.startswith('data:'):
                    import json
                    try:
                        json_data = json.loads(decoded_line[len('data: '):])
                        if json_data.get('event') == 'agent_message':
                            final_answer += json_data.get('answer', '')
                    except json.JSONDecodeError:
                        continue

        return jsonify({"text": final_answer})

    except requests.exceptions.RequestException as e:
        return jsonify({"text": f"Error calling Dify agent: {str(e)}"})

@app.route('/static/generated_files/<filename>')
def serve_generated_file(filename):
    return send_from_directory(GENERATED_FILES_DIR, filename)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)

```
