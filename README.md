# mp4-video-
from flask import Flask, render_template_string, request, send_file
from werkzeug.utils import secure_filename
import os
import cv2
import numpy as np

app = Flask(__name__)
UPLOAD_FOLDER = 'uploads'
OUTPUT_FOLDER = 'outputs'
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['OUTPUT_FOLDER'] = OUTPUT_FOLDER

# واجهة الموقع (HTML مباشرة داخل الكود لتسهيل التشغيل)
@app.route('/')
def home():
    return '''
    <!DOCTYPE html>
    <html lang="ar" dir="rtl">
    <head>
        <meta charset="UTF-8">
        <title>منصة تعديل وفلترة الفيديو</title>
        <style>
            body { font-family: Tahoma, sans-serif; background: #121212; color: #fff; text-align: center; padding-top: 40px; }
            form { background: #1e1e1e; display: inline-block; padding: 25px; border-radius: 12px; box-shadow: 0 0 15px rgba(0,0,0,0.6); width: 320px; }
            input, button { margin: 12px 0; padding: 10px; width: 90%; border-radius: 6px; border: none; }
            input[type="file"] { background: #2a2a2a; color: #fff; }
            input[type="range"] { background: transparent; }
            button { background: #4CAF50; color: white; font-weight: bold; cursor: pointer; font-size: 16px; }
            button:hover { background: #45a049; }
            label { font-size: 14px; color: #ccc; display: block; text-align: right; }
        </style>
    </head>
    <body>
        <h1>تطبيق معالجة الفيديو بالبرمجة</h1>
        <form method="POST" action="/process" enctype="multipart/form-data">
            <label>اختر ملف الفيديو:</label>
            <input type="file" name="video" required>
            
            <label>السطوع (Brightness):</label>
            <input type="range" name="brightness" min="0.5" max="1.5" step="0.1" value="1">
            
            <label>التباين (Contrast):</label>
            <input type="range" name="contrast" min="0.5" max="1.5" step="0.1" value="1">
            
            <button type="submit">معالجة وتنزيل الفيديو</button>
        </form>
    </body>
    </html>
    '''

# مسار معالجة الفيديو
@app.route('/process', methods=['POST'])
def process():
    if 'video' not in request.files:
        return "لم يتم العثور على ملف الفيديو!", 400
    
    video = request.files['video']
    if video.filename == '':
        return "اسم الملف فارغ!", 400

    brightness = float(request.form.get('brightness', 1.0))
    contrast = float(request.form.get('contrast', 1.0))
    
    filename = secure_filename(video.filename)
    input_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
    output_path = os.path.join(app.config['OUTPUT_FOLDER'], f"processed_{filename}")
    
    video.save(input_path)
    
    # قراءة الفيديو ومعالجته باستخدام OpenCV
    cap = cv2.VideoCapture(input_path)
    fps = int(cap.get(cv2.CAP_PROP_FPS)) or 25
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
    
    fourcc = cv2.VideoWriter_fourcc(*'mp4v')
    out = cv2.VideoWriter(output_path, fourcc, fps, (width, height))
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # تطبيق الفلاتر (السطوع والتباين)
        alpha = contrast
        beta = (brightness - 1.0) * 50
        processed_frame = cv2.convertScaleAbs(frame, alpha=alpha, beta=beta)
        
        out.write(processed_frame)
        
    cap.release()
    out.release()
    
    return send_file(output_path, as_attachment=True)

if __name__ == '__main__':
    os.makedirs(UPLOAD_FOLDER, exist_ok=True)
    os.makedirs(OUTPUT_FOLDER, exist_ok=True)
    app.run(host='0.0.0.0', port=5000, debug=True)
