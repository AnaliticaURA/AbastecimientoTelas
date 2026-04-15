"""
URA Abastecimiento — Servidor Flask
Expone una API mínima para que el frontend lea y actualice los datos.
"""
import json
import os
from pathlib import Path
from flask import Flask, jsonify, render_template, request

# Importar motor de análisis
from analysis import run_analysis

app = Flask(__name__)

DATA_DIR   = Path(__file__).parent / 'data'
CACHE_FILE = DATA_DIR / 'analysis_output.json'

# Nombres de archivo esperados (pueden cambiarse vía variables de entorno)
REQ_FILENAME = os.getenv('REQ_FILENAME', 'Requerimiento_telas.xlsx')
INV_FILENAME = os.getenv('INV_FILENAME', 'Inventario_telas.xlsx')


def get_excel_paths():
    req_path = DATA_DIR / REQ_FILENAME
    inv_path = DATA_DIR / INV_FILENAME
    return req_path, inv_path


def file_status():
    req_path, inv_path = get_excel_paths()
    return {
        'req': {'name': REQ_FILENAME, 'exists': req_path.exists()},
        'inv': {'name': INV_FILENAME, 'exists': inv_path.exists()},
    }


# ─── RUTAS ────────────────────────────────────────────────────────────────────

@app.route('/')
def index():
    return render_template('index.html')


@app.route('/api/status')
def api_status():
    """Estado de los archivos y si hay caché disponible."""
    fs = file_status()
    return jsonify({
        'files': fs,
        'cache_exists': CACHE_FILE.exists(),
        'ready': fs['req']['exists'] and fs['inv']['exists'],
    })


@app.route('/api/analyse', methods=['POST'])
def api_analyse():
    """Lanza el análisis y devuelve (y cachea) el resultado."""
    req_path, inv_path = get_excel_paths()

    if not req_path.exists():
        return jsonify({'error': f'Archivo no encontrado: {REQ_FILENAME}'}), 400
    if not inv_path.exists():
        return jsonify({'error': f'Archivo no encontrado: {INV_FILENAME}'}), 400

    try:
        result = run_analysis(str(req_path), str(inv_path))
        CACHE_FILE.write_text(
            json.dumps(result, ensure_ascii=False, default=str)
        )
        return jsonify({'ok': True, 'summary': result['summary']})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/api/data')
def api_data():
    """Devuelve el último análisis cacheado."""
    if not CACHE_FILE.exists():
        return jsonify({'error': 'No hay análisis disponible. Sube los archivos y ejecuta el análisis.'}), 404
    data = json.loads(CACHE_FILE.read_text())
    return jsonify(data)


@app.route('/api/upload', methods=['POST'])
def api_upload():
    """Sube un archivo Excel (req o inv) al directorio data/."""
    file_type = request.form.get('type')  # 'req' o 'inv'
    if file_type not in ('req', 'inv'):
        return jsonify({'error': 'type debe ser req o inv'}), 400

    f = request.files.get('file')
    if not f:
        return jsonify({'error': 'No se recibió archivo'}), 400

    dest = DATA_DIR / (REQ_FILENAME if file_type == 'req' else INV_FILENAME)
    DATA_DIR.mkdir(exist_ok=True)
    f.save(str(dest))
    return jsonify({'ok': True, 'saved': dest.name})


if __name__ == '__main__':
    DATA_DIR.mkdir(exist_ok=True)
    app.run(debug=True, port=5000)
