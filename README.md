<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PatternScan QC VISION (Robust Engine)</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600&family=Oxanium:wght@700&display=swap');
        
        :root { 
            --primary: #0284c7; 
            --success: #0d9488; 
            --danger: #ef4444; 
            --bg: #f1f5f9; 
        }
        
        body { 
            font-family: 'Inter', sans-serif; 
            background: var(--bg); 
            padding: 16px; 
            margin: 0; 
        }
        
        .panel { 
            background: #ffffff; 
            border: 1px solid #e2e8f0; 
            border-radius: 14px; 
            padding: 16px; 
            margin-bottom: 16px; 
        }
        
        .panel-title { 
            font-size: 0.75rem; 
            color: #64748b; 
            font-weight: 700; 
            margin-bottom: 12px; 
            font-family: 'Oxanium', sans-serif; 
        }
        
        .display-box { 
            border: 1px dashed #cbd5e1; 
            border-radius: 10px; 
            aspect-ratio: 4/3; 
            display: flex; 
            align-items: center; 
            justify-content: center; 
            overflow: hidden; 
            position: relative; 
            background-color: #fafafa;
        }
        
        .display-box img { 
            width: 100%; 
            height: 100%; 
            object-fit: contain; 
        }
        
        .file-input-wrapper {
            margin-top: 12px;
            width: 100%;
        }
        
        .btn-primary { 
            background: var(--primary); 
            color: white; 
            border: none; 
            width: 100%; 
            padding: 14px; 
            border-radius: 12px; 
            font-family: 'Oxanium', sans-serif; 
            font-size: 1rem;
            font-weight: 700; 
            cursor: pointer; 
            transition: 0.2s;
        }
        
        .btn-primary:disabled { 
            background: #cbd5e1; 
            cursor: not-allowed;
        }
        
        .result-card { 
            padding: 16px; 
            border-radius: 10px; 
            text-align: center; 
            margin-top: 16px; 
            display: none; 
            font-size: 0.9rem;
        }
    </style>
    
    <!-- โหลด OpenCV ผ่าน CDN ทางการ -->
    <script async src="https://docs.opencv.org/4.x/opencv.js" type="text/javascript"></script>
</head>
<body>

    <div class="panel">
        <div class="panel-title">1. รูปแบบกลุ่มจุดอ้างอิง (REFERENCE)</div>
        <div class="display-box"><img id="refImage"></div>
        <input type="file" id="fileInput" class="file-input-wrapper" accept="image/*" capture="environment">
    </div>

    <div class="panel">
        <div class="panel-title">2. ภาพตรวจทานหน้างาน (TARGET SCAN)</div>
        <div class="display-box"><img id="targetImage"></div>
        <input type="file" id="fileInputTarget" class="file-input-wrapper" accept="image/*" capture="environment">
    </div>

    <button class="btn-primary" id="btnScan" disabled>กำลังโหลดเอนจิน...</button>

    <div id="resultCard" class="result-card">
        <div id="resultText" style="font-weight: 700;"></div>
    </div>

    <script>
        const refImage = document.getElementById('refImage');
        const targetImage = document.getElementById('targetImage');
        const btnScan = document.getElementById('btnScan');
        const resultCard = document.getElementById('resultCard');
        const resultText = document.getElementById('resultText');

        let isReady = false;

        function onOpenCvReady() {
            if (isReady) return;
            isReady = true;
            btnScan.innerText = "ตรวจสอบจุด (พร้อมใช้งาน)";
            checkReady();
        }

        // ระบบตรวจสอบความพร้อมแบบวนลูป
        let checkInterval = setInterval(() => {
            if (typeof cv !== 'undefined' && typeof cv.Mat === 'function') {
                clearInterval(checkInterval);
                onOpenCvReady();
            }
        }, 300);

        // ระบบตัดรอบแจ้งเตือนหากโหลดเกิน 12 วินาที
        setTimeout(() => {
            if (!isReady) {
                clearInterval(checkInterval);
                btnScan.innerText = "❌ โหลดเอนจินไม่สำเร็จ (ถูกบล็อก/ไม่มีเน็ต)";
                btnScan.style.background = "var(--danger)";
            }
        }, 12000);

        document.getElementById('fileInput').onchange = (e) => { 
            if(e.target.files[0]) {
                refImage.src = URL.createObjectURL(e.target.files[0]); 
                checkReady(); 
            }
        };
        
        document.getElementById('fileInputTarget').onchange = (e) => { 
            if(e.target.files[0]) {
                targetImage.src = URL.createObjectURL(e.target.files[0]); 
                checkReady(); 
            }
        };

        function checkReady() { 
            if (isReady && refImage.src && targetImage.src) {
                btnScan.disabled = false;
            } else {
                btnScan.disabled = true;
            }
        }

        function getResizedMat(imgElement) {
            let canvas = document.createElement('canvas');
            let origW = imgElement.naturalWidth || imgElement.width || 320;
            let origH = imgElement.naturalHeight || imgElement.height || 240;
            
            let maxDim = 1024;
            let w = origW;
            let h = origH;
            
            if (w > maxDim || h > maxDim) {
                if (w > h) {
                    h = Math.round((h * maxDim) / w);
                    w = maxDim;
                } else {
                    w = Math.round((w * maxDim) / h);
                    h = maxDim;
                }
            }

            canvas.width = w; 
            canvas.height = h;
            
            let ctx = canvas.getContext('2d');
            ctx.fillStyle = '#ffffff'; 
            ctx.fillRect(0, 0, w, h);
            ctx.drawImage(imgElement, 0, 0, w, h);
            
            let src = cv.imread(canvas);
            return src;
        }

        function getDots(imgElement) {
            let mat = getResizedMat(imgElement);
            let rgb = new cv.Mat();
            cv.cvtColor(mat, rgb, cv.COLOR_RGBA2RGB);

            let filtered = new cv.Mat();
            cv.bilateralFilter(rgb, filtered, 9, 75, 75);

            let hsv = new cv.Mat();
            cv.cvtColor(filtered, hsv, cv.COLOR_RGB2HSV);
            
            let hsvPlanes = new cv.MatVector();
            cv.split(hsv, hsvPlanes);
            
            let vChannel = hsvPlanes.get(2);
            let clahe = new cv.CLAHE(2.0, new cv.Size(8, 8)); 
            clahe.apply(vChannel, vChannel);
            
            hsvPlanes.set(2, vChannel);
            cv.merge(hsvPlanes, hsv);
            
            let mask = new cv.Mat();
            let low = new cv.Mat(hsv.rows, hsv.cols, hsv.type(), [75, 50, 50, 0]);
            let high = new cv.Mat(hsv.rows, hsv.cols, hsv.type(), [110, 255, 255, 255]);
            cv.inRange(hsv, low, high, mask);

            let kernel = cv.getStructuringElement(cv.MORPH_RECT, new cv.Size(3, 3));
            cv.morphologyEx(mask, mask, cv.MORPH_OPEN, kernel);
            
            let contours = new cv.MatVector();
            cv.findContours(mask, contours, new cv.Mat(), cv.RETR_EXTERNAL, cv.CHAIN_APPROX_SIMPLE);
            
            let pts = [];
            let imgArea = mat.cols * mat.rows; 
            
            for (let i = 0; i < contours.size(); ++i) {
                let rect = cv.boundingRect(contours.get(i));
                let area = rect.width * rect.height;
                
                if (area >= (imgArea * 0.000002) && area <= (imgArea * 0.08)) { 
                    pts.push({
                        x: rect.x + rect.width / 2, 
                        y: rect.y + rect.height / 2,
                        width: mat.cols 
                    });
                }
            }
            
            mat.delete(); rgb.delete(); filtered.delete(); hsv.delete(); 
            hsvPlanes.delete(); vChannel.delete(); clahe.delete();
            mask.delete(); low.delete(); high.delete(); kernel.delete(); contours.delete();
            
            return pts;
        }

        btnScan.onclick = () => {
            resultCard.style.display = 'none';
            btnScan.innerText = "กำลังประมวลผล...";
            btnScan.disabled = true;

            setTimeout(() => {
                try {
                    let p1 = getDots(refImage);
                    let p2 = getDots(targetImage);
                    
                    resultCard.style.display = 'block';
                    
                    if (p1.length !== p2.length || p1.length === 0) {
                        resultText.innerText = `FAIL // จำนวนจุดไม่เท่ากัน (อ้างอิง: ${p1.length} | สแกน: ${p2.length})`;
                        resultCard.style.background = 'var(--danger)'; 
                        resultCard.style.color = 'white';
                        btnScan.innerText = "ตรวจสอบจุด (พร้อมใช้งาน)";
                        btnScan.disabled = false;
                        return;
                    }

                    let matchCount = 0;
                    let tolerance = (p1[0].width || 1000) * 0.07; 

                    p1.forEach(pt1 => {
                        let matched = false;
                        p2.forEach(pt2 => {
                            if (!matched) {
                                let dist = Math.hypot(pt1.x - pt2.x, pt1.y - pt2.y);
                                if (dist <= tolerance) {
                                    matchCount++;
                                    matched = true;
                                }
                            }
                        });
                    });

                    if (matchCount === p1.length) {
                        resultText.innerText = `PASS 100%`;
                        resultCard.style.background = 'var(--success)'; 
                        resultCard.style.color = 'white';
                    } else {
                        resultText.innerText = `FAIL // ตำแหน่งคลาดเคลื่อน (เจอ ${matchCount} จาก ${p1.length} จุด)`;
                        resultCard.style.background = 'var(--danger)'; 
                        resultCard.style.color = 'white';
                    }

                } catch (err) {
                    console.error("OpenCV Error:", err);
                    alert("เกิดข้อผิดพลาดในการประมวลผล");
                }

                btnScan.innerText = "ตรวจสอบจุด (พร้อมใช้งาน)";
                btnScan.disabled = false;
            }, 100);
        };
    </script>
</body>
</html>
