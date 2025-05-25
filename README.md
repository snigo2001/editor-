import os
from datetime import datetime
from flask import Flask, request, render_template_string, send_from_directory, flash, redirect, url_for
from werkzeug.utils import secure_filename
from moviepy.editor import VideoFileClip

# ---------------------
# CONFIGURATION
# ---------------------
UPLOAD_FOLDER = "uploads"
OUTPUT_FOLDER = "processed"
ALLOWED_EXTENSIONS = {"mp4"}

app = Flask(__name__)
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER
app.config["OUTPUT_FOLDER"] = OUTPUT_FOLDER
app.secret_key = "super-secret-key"  # needed for flash messages

os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(OUTPUT_FOLDER, exist_ok=True)

# ---------------------
# UTILITY FUNCTIONS
# ---------------------

def allowed_file(filename: str) -> bool:
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS


def cut_video(source_path: str, start: float, end: float) -> str:
    """Cuts the video from start to end seconds using moviepy and returns output path."""
    filename = os.path.basename(source_path)
    name, ext = os.path.splitext(filename)
    timestamp = datetime.now().strftime("%Y%m%d%H%M%S")
    output_name = f"{name}_CUT_{timestamp}{ext}"
    output_path = os.path.join(app.config["OUTPUT_FOLDER"], output_name)

    with VideoFileClip(source_path) as clip:
        end = min(end, clip.duration)  # guard against values > duration
        subclip = clip.subclip(start, end)
        subclip.write_videofile(output_path, codec="libx264", audio_codec="aac")

    return output_name  # return relative name for URL building


# ---------------------
# ROUTES
# ---------------------
INDEX_HTML = """
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Quick Video Editor IA</title>
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
      rel="stylesheet"
    />
  </head>
  <body class="bg-dark text-light">
    <div class="container py-4">
      <h1 class="mb-4 text-center">Editor de Video Rápido con IA (Demo)</h1>

      <!-- Upload form -->
      <div class="card bg-secondary mb-4 p-3">
        <form
          class="row g-3 align-items-end"
          action="{{ url_for('upload') }}"
          method="post"
          enctype="multipart/form-data"
        >
          <div class="col-md-6">
            <label for="file" class="form-label">Subir video (.mp4)</label>
            <input class="form-control" type="file" id="file" name="file" required />
          </div>
          <div class="col-md-2">
            <label for="start" class="form-label">Inicio (seg)</label>
            <input
              type="number"
              step="0.01"
              min="0"
              class="form-control"
              id="start"
              name="start"
              value="0"
              required
            />
          </div>
          <div class="col-md-2">
            <label for="end" class="form-label">Fin (seg)</label>
            <input
              type="number"
              step="0.01"
              min="0"
              class="form-control"
              id="end"
              name="end"
              value="10"
              required
            />
          </div>
          <div class="col-md-2 d-grid">
            <button type="submit" class="btn btn-primary">Procesar</button>
          </div>
        </form>
      </div>

      {% with messages = get_flashed_messages() %}
      {% if messages %}
      <div class="alert alert-warning">
        {% for message in messages %}
        <div>{{ message }}</div>
        {% endfor %}
      </div>
      {% endif %}
      {% endwith %}

      {% if original or processed %}
      <div class="row g-4">
        {% if original %}
        <div class="col-md-6">
          <h4>Original</h4>
          <video controls style="width: 100%" src="{{ url_for('serve_file', folder='uploads', filename=original) }}"></video>
        </div>
        {% endif %}

        {% if processed %}
        <div class="col-md-6">
          <h4>Procesado</h4>
          <video controls style="width: 100%" src="{{ url_for('serve_file', folder='processed', filename=processed) }}"></video>
        </div>
        {% endif %}
      </div>
      {% endif %}
    </div>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
  </body>
</html>
"""


@app.route("/")
def index():
    """Home: show upload form and (optionally) videos."""
    original = request.args.get("original")
    processed = request.args.get("processed")
    return render_template_string(INDEX_HTML, original=original, processed=processed)


@app.route("/upload", methods=["POST"])
def upload():
    file = request.files.get("file")
    if not file or file.filename == "":
        flash("Debe seleccionar un archivo.")
        return redirect(url_for("index"))

    if not allowed_file(file.filename):
        flash("Formato no permitido. Solo .mp4")
        return redirect(url_for("index"))

    filename = secure_filename(file.filename)
    save_path = os.path.join(app.config["UPLOAD_FOLDER"], filename)
    file.save(save_path)

    # Parse cut times
    try:
        start = float(request.form.get("start", 0))
        end = float(request.form.get("end", 10))
    except ValueError:
        flash("Valores de inicio/fin no válidos")
        return redirect(url_for("index"))

    if end <= start:
        flash("El tiempo de fin debe ser mayor que el de inicio")
        return redirect(url_for("index"))

    # Cut video
    try:
        output_name = cut_video(save_path, start, end)
    except Exception as e:
        flash(f"Error procesando video: {e}")
        return redirect(url_for("index"))

    return redirect(url_for("index", original=filename, processed=output_name))


@app.route("/media/<folder>/<path:filename>")
def serve_file(folder, filename):
    if folder not in {"uploads", "processed"}:
        return "Carpeta no permitida", 404
    directory = app.config["UPLOAD_FOLDER"] if folder == "uploads" else app.config["OUTPUT_FOLDER"]
    return send_from_directory(directory, filename, as_attachment=False)


if __name__ == "__main__":
    app.run(debug=True, port=5000)
