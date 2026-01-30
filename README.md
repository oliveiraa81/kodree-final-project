# kodree-final-project
Local OCR

Here is the complete source code for LocalExtract.
Instructions
Install Tesseract OCR: Ensure Tesseract is installed on the machine.
Windows: Default path assumed is C:\Program Files\Tesseract-OCR\tesseract.exe.
Folder Structure: Create a folder named LocalExtract and arrange the files as follows:
code
Text
LocalExtract/
├── static/
│   └── style.css
├── templates/
│   └── index.html
├── app.py
├── requirements.txt
└── run_app.bat
1. requirements.txt
code
Text
flask
pytesseract
pandas
openpyxl
pillow
werkzeug
2. app.py
This file handles the OCR processing and the specific Regex logic required for the Portuguese freight orders.
code
Python
import os
import re
import io
import pytesseract
import pandas as pd
from PIL import Image
from flask import Flask, render_template, request, send_file, jsonify
from werkzeug.utils import secure_filename

app = Flask(__name__)

# CONFIGURATION
# IMPORTANT: Update this path if Tesseract is installed elsewhere
# Common Windows path: r'C:\Program Files\Tesseract-OCR\tesseract.exe'
# Common Linux/Mac path: '/usr/bin/tesseract'
if os.name == 'nt':
    pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

UPLOAD_FOLDER = 'temp_uploads'
if not os.path.exists(UPLOAD_FOLDER):
    os.makedirs(UPLOAD_FOLDER)

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

def safe_extract(pattern, text, group_index=1, default=""):
    """Helper to safely extract regex matches without crashing."""
    try:
        match = re.search(pattern, text, re.IGNORECASE | re.MULTILINE)
        if match:
            return match.group(group_index).strip()
    except Exception:
        pass
    return default

def parse_ocr_text(text):
    """
    Parses raw OCR text into the specific 23 columns defined in the PRD.
    Logic is tailored for Portuguese Transport Orders.
    """
    data = {}
    
    # 1. Data de Carga
    data['Data de Carga'] = safe_extract(r'Data de Carga\s*[:\.]?\s*(\d{2}[-/]\d{2}[-/]\d{4})', text)

    # 2 & 3. Origem (País & Localidade)
    # Looking for: "Local de origem: PT - 47 Jesufrei"
    origem_line = safe_extract(r'Local de origem\s*[:\.]?\s*(.*)', text, 1)
    if origem_line and '-' in origem_line:
        parts = origem_line.split('-', 1)
        data['Origem País'] = parts[0].strip()
        data['Origem Localidade'] = parts[1].strip()
    else:
        data['Origem País'] = ""
        data['Origem Localidade'] = origem_line

    # 4 & 5. Destino (País & Localidade)
    destino_line = safe_extract(r'Local de Destino\s*[:\.]?\s*(.*)', text, 1)
    if destino_line and '-' in destino_line:
        parts = destino_line.split('-', 1)
        data['Destino País'] = parts[0].strip()
        data['Destino Localidade'] = parts[1].strip()
    else:
        data['Destino País'] = ""
        data['Destino Localidade'] = destino_line

    # 6. Peso Bruto (kg) - Extract digits/floats before 'kg'
    data['Peso Bruto (kg)'] = safe_extract(r'Peso bruto\s*[:\.]?\s*([\d\.,\s]+)', text).replace(' ', '')

    # 7. Tipo de Serviço - Extract text inside parentheses in Peso bruto line
    # Ex: Peso bruto: ... (FTL - Carga Completa)
    data['Tipo de Serviço'] = safe_extract(r'Peso bruto.*?\((.*?)\)', text)

    # 8. Quantidade (m estrado)
    data['Quantidade (m estrado)'] = safe_extract(r'Quantidade\s*[:\.]?\s*([\d\.,]+)', text)

    # 9. Tipo de Camião
    data['Tipo de Camião'] = safe_extract(r'(?:Tipo de Camião|Estrutura)\s*[:\.]?\s*(.*)', text)

    # 10. Tipo de Carga / Equipamento
    data['Tipo de Carga'] = safe_extract(r'Equipamento\s*[:\.]?\s*(.*)', text)

    # 11. Data de Descarga (Look inside Observações or general text for "Descarga DD/MM/YYYY")
    data['Data de Descarga'] = safe_extract(r'Descarga\s.*?(\d{2}[-/]\d{2}[-/]\d{4})', text)

    # 12. Janela de Descarga (Look for HHhMM pattern)
    data['Janela de Descarga'] = safe_extract(r'(\d{1,2}h\d{2})', text, 1)

    # 13. Observações (Capture block)
    # Simple strategy: Find header and take next 200 chars or until next header
    obs_match = re.search(r'(?:Observações|Notas Importantes)([\s\S]{1,500})', text, re.IGNORECASE)
    data['Observações'] = obs_match.group(1).strip() if obs_match else ""

    # 14. Instruções Adicionais
    data['Instruções Adicionais'] = safe_extract(r'(?:Instruções|Requisitos)([\s\S]{1,200})', text)

    # 15. Ref / Assunto Email
    data['Ref / Assunto Email'] = safe_extract(r'(?:Ref|Assunto).*?[:\.]?\s*(.*)', text)

    # 16 & 17. Empresa & Morada (REMETENTE Context)
    # Assumes "REMETENTE" is followed by Company on line 1, Address on line 2
    remetente_block = re.search(r'REMETENTE\s*[\n\r]+(.*?)(?:[\n\r]+(.*?))?[\n\r]', text, re.MULTILINE)
    if remetente_block:
        data['Empresa'] = remetente_block.group(1).strip()
        data['Remetente Morada'] = remetente_block.group(2).strip() if remetente_block.group(2) else ""
    else:
        data['Empresa'] = ""
        data['Remetente Morada'] = ""

    # 18. Telefone
    data['Telefone'] = safe_extract(r'(?:Telefone|Telemóvel|Contactos).*?([\+\d\s]{9,})', text)

    # 19. Emails
    data['Emails'] = safe_extract(r'([a-zA-Z0-9._-]+@[a-zA-Z0-9._-]+\.[a-zA-Z0-9_-]+)', text, 1)

    # 20. Estado da Oportunidade
    data['Estado da Oportunidade'] = ""

    # 21. Valor Estimado
    data['Valor Estimado'] = safe_extract(r'Valor\s*[:\.]?\s*([\d\.,]+)', text)

    # 22. Responsável Interno
    data['Responsável Interno'] = ""

    # 23. Notas Internas
    data['Notas Internas'] = ""

    return data

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
        return jsonify({'error': 'No file part'}), 400
    
    file = request.files['file']
    if file.filename == '':
        return jsonify({'error': 'No selected file'}), 400

    if file:
        filename = secure_filename(file.filename)
        filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        file.save(filepath)

        try:
            # 1. OCR Processing
            image = Image.open(filepath)
            # Portuguese language hint 'por' matches best, but 'eng' usually works for structure
            # If tesseract-por is installed, change lang='por'
            raw_text = pytesseract.image_to_string(image) 
            
            # 2. Parse Data
            parsed_data = parse_ocr_text(raw_text)

            # 3. Create Excel
            df = pd.DataFrame([parsed_data])
            output = io.BytesIO()
            with pd.ExcelWriter(output, engine='openpyxl') as writer:
                df.to_excel(writer, index=False, sheet_name='Extracted Data')
            output.seek(0)

            # Cleanup
            image.close()
            os.remove(filepath)

            return send_file(
                output,
                as_attachment=True,
                download_name=f"extracted_{os.path.splitext(filename)[0]}.xlsx",
                mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
            )

        except Exception as e:
            return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(debug=True, port=5000)
3. templates/index.html
A single-file HTML/JS frontend with drag-and-drop logic.
code
Html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LocalExtract | Offline OCR</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>

    <div class="app-container">
        <header>
            <h1>LocalExtract</h1>
            <p>Secure Offline OCR & Data Parsing</p>
        </header>

        <main>
            <div id="drop-zone" class="drop-zone">
                <div class="icon-container">
                    <!-- SVG Icon for Upload -->
                    <svg width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="#BB86FC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
                        <polyline points="17 8 12 3 7 8"></polyline>
                        <line x1="12" y1="3" x2="12" y2="15"></line>
                    </svg>
                </div>
                <p class="main-text">Drag & Drop Transport Order Here</p>
                <p class="sub-text">or click to browse (.png, .jpg, .jpeg)</p>
                <input type="file" id="file-input" accept=".png, .jpg, .jpeg" hidden>
            </div>

            <div id="status-container" class="hidden">
                <div class="spinner"></div>
                <p id="status-text">Extracting data via Tesseract...</p>
            </div>

            <div id="result-container" class="hidden">
                <div class="success-icon">✓</div>
                <p>Extraction Complete!</p>
                <a id="download-link" href="#" class="btn-download">Download Excel (.xlsx)</a>
                <button id="reset-btn" class="btn-text">Process Another File</button>
            </div>
        </main>
        
        <footer>
            <p>Running Locally | No Internet Connection Required</p>
        </footer>
    </div>

    <script>
        const dropZone = document.getElementById('drop-zone');
        const fileInput = document.getElementById('file-input');
        const statusContainer = document.getElementById('status-container');
        const resultContainer = document.getElementById('result-container');
        const downloadLink = document.getElementById('download-link');
        const resetBtn = document.getElementById('reset-btn');

        // Drag and Drop Events
        ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
            dropZone.addEventListener(eventName, preventDefaults, false);
        });

        function preventDefaults(e) {
            e.preventDefault();
            e.stopPropagation();
        }

        ['dragenter', 'dragover'].forEach(eventName => {
            dropZone.addEventListener(eventName, highlight, false);
        });

        ['dragleave', 'drop'].forEach(eventName => {
            dropZone.addEventListener(eventName, unhighlight, false);
        });

        function highlight(e) {
            dropZone.classList.add('highlight');
        }

        function unhighlight(e) {
            dropZone.classList.remove('highlight');
        }

        dropZone.addEventListener('drop', handleDrop, false);
        dropZone.addEventListener('click', () => fileInput.click());
        fileInput.addEventListener('change', (e) => handleFiles(e.target.files));

        function handleDrop(e) {
            const dt = e.dataTransfer;
            const files = dt.files;
            handleFiles(files);
        }

        function handleFiles(files) {
            if (files.length > 0) {
                uploadFile(files[0]);
            }
        }

        function uploadFile(file) {
            // Validation
            const validTypes = ['image/jpeg', 'image/png', 'image/jpg'];
            if (!validTypes.includes(file.type)) {
                alert('Invalid file type. Please upload an image.');
                return;
            }

            // UI Updates
            dropZone.classList.add('hidden');
            statusContainer.classList.remove('hidden');
            resultContainer.classList.add('hidden');

            const formData = new FormData();
            formData.append('file', file);

            fetch('/upload', {
                method: 'POST',
                body: formData
            })
            .then(response => {
                if (response.ok) {
                    return response.blob();
                }
                throw new Error('Processing failed');
            })
            .then(blob => {
                // Create Download Link
                const url = window.URL.createObjectURL(blob);
                downloadLink.href = url;
                downloadLink.download = `Parsed_${file.name.split('.')[0]}.xlsx`;
                
                statusContainer.classList.add('hidden');
                resultContainer.classList.remove('hidden');
            })
            .catch(error => {
                alert('Error processing file: ' + error.message);
                dropZone.classList.remove('hidden');
                statusContainer.classList.add('hidden');
            });
        }

        resetBtn.addEventListener('click', () => {
            resultContainer.classList.add('hidden');
            dropZone.classList.remove('hidden');
            fileInput.value = '';
        });
    </script>
</body>
</html>
4. static/style.css
Dark mode styling using CSS variables.
code
CSS
:root {
    --bg-color: #121212;
    --surface-color: #1E1E1E;
    --text-primary: #E0E0E0;
    --text-secondary: #A0A0A0;
    --accent-color: #BB86FC;
    --accent-hover: #9965f4;
    --error-color: #CF6679;
    --success-color: #03DAC6;
}

body {
    background-color: var(--bg-color);
    color: var(--text-primary);
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    margin: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
}

.app-container {
    width: 100%;
    max-width: 600px;
    padding: 2rem;
    text-align: center;
}

header h1 {
    color: var(--accent-color);
    margin-bottom: 0.5rem;
}

header p {
    color: var(--text-secondary);
    margin-top: 0;
    margin-bottom: 2rem;
}

.drop-zone {
    background-color: var(--surface-color);
    border: 2px dashed var(--text-secondary);
    border-radius: 12px;
    padding: 3rem;
    cursor: pointer;
    transition: all 0.3s ease;
}

.drop-zone:hover, .drop-zone.highlight {
    border-color: var(--accent-color);
    background-color: #1e1e1e; /* Slight tint logic handled by opacity usually, direct color here */
    box-shadow: 0 0 15px rgba(187, 134, 252, 0.1);
}

.drop-zone .main-text {
    font-size: 1.2rem;
    font-weight: bold;
    margin: 1rem 0 0.5rem 0;
}

.drop-zone .sub-text {
    color: var(--text-secondary);
    font-size: 0.9rem;
}

/* Spinner */
.spinner {
    border: 4px solid rgba(255, 255, 255, 0.1);
    width: 40px;
    height: 40px;
    border-radius: 50%;
    border-left-color: var(--accent-color);
    animation: spin 1s linear infinite;
    margin: 0 auto 1rem auto;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.hidden {
    display: none;
}

/* Results */
#result-container {
    background-color: var(--surface-color);
    padding: 2rem;
    border-radius: 12px;
    border: 1px solid var(--success-color);
}

.success-icon {
    font-size: 3rem;
    color: var(--success-color);
    margin-bottom: 1rem;
}

.btn-download {
    display: inline-block;
    background-color: var(--accent-color);
    color: #000;
    padding: 0.8rem 2rem;
    border-radius: 25px;
    text-decoration: none;
    font-weight: bold;
    margin-top: 1rem;
    transition: background-color 0.2s;
}

.btn-download:hover {
    background-color: var(--accent-hover);
}

.btn-text {
    display: block;
    margin: 1.5rem auto 0 auto;
    background: none;
    border: none;
    color: var(--text-secondary);
    text-decoration: underline;
    cursor: pointer;
}

.btn-text:hover {
    color: var(--text-primary);
}

footer {
    margin-top: 3rem;
    color: var(--text-secondary);
    font-size: 0.8rem;
    opacity: 0.5;
}
5. run_app.bat
A Windows batch script to set up and run the application.
code
Batch
@echo off
TITLE LocalExtract - Offline OCR App
COLOR 0A

echo =====================================================
echo    LocalExtract: Transport Order OCR Automation
echo =====================================================
echo.

:: Check if Python is installed
python --version >nul 2>&1
IF %ERRORLEVEL% NEQ 0 (
    echo [ERROR] Python is not installed or not in PATH.
    echo Please install Python 3.x and try again.
    pause
    exit
)

echo [1/3] Checking requirements...
pip install -r requirements.txt
echo.

echo [2/3] Checking Tesseract path...
if exist "C:\Program Files\Tesseract-OCR\tesseract.exe" (
    echo [OK] Tesseract found at default location.
) else (
    echo [WARNING] Tesseract not found at default location.
    echo Please ensure you update the path in 'app.py' if installed elsewhere.
)
echo.

echo [3/3] Starting Local Server...
echo Open your browser to: http://127.0.0.1:5000
echo.
python app.py

pause
