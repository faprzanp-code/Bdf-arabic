<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>قارئ PDF عربي صوتي ذكي</title>
    
    <!-- استدعاء مكتبة Tesseract للتعرف الضوئي على النصوص العربية (OCR) -->
    <script src="https://jsdelivr.net"></script>

    <style>
        body { 
            margin: 0; 
            padding: 15px; 
            background: #f3f3f3; 
            font-family: Arial, sans-serif; 
        } 
        .box { 
            max-width: 750px; 
            margin: auto; 
            background: white; 
            padding: 20px; 
            border-radius: 18px; 
            box-shadow: 0 3px 15px #0002; 
        } 
        h1 { text-align: center; color: #222; } 
        input { width: 100%; font-size: 17px; margin: 15px 0; box-sizing: border-box; padding: 10px; } 
        textarea { 
            width: 100%; 
            height: 320px; 
            box-sizing: border-box; 
            font-size: 19px; 
            line-height: 1.8; 
            padding: 12px; 
            border: 1px solid #ccc;
            border-radius: 8px;
        } 
        button { 
            padding: 12px 15px; 
            margin: 5px; 
            border: 0; 
            border-radius: 10px; 
            background: #222; 
            color: white; 
            font-size: 17px; 
            cursor: pointer;
            transition: background 0.2s;
        } 
        button:hover { background: #444; }
        #status { 
            padding: 12px; 
            background: #eee; 
            border-radius: 10px; 
            margin: 10px 0; 
            font-weight: bold;
            color: #333;
        } 
    </style>
</head>
<body>

<div class="box">
    <h1>📖 قارئ PDF عربي صوتي</h1>
    
    <label for="pdfFile">اختار كتاب PDF:</label>
    <input type="file" id="pdfFile" accept=".pdf">

    <div id="status">ℹ️ يرجى اختيار ملف PDF للبدء...</div>

    <textarea id="text" placeholder="النص المستخرج سيظهر هنا تلقائياً، ويمكنك تعديله قبل القراءة..."></textarea>
    
    <div style="text-align: center; margin-top: 10px;">
        <button onclick="readText()">🔊 اقرأ</button>
        <button onclick="pauseText()">⏸️ توقف مؤقت</button>
        <button onclick="resumeText()">▶️ متابعة</button>
        <button onclick="stopText()">⏹️ إيقاف</button>
    </div>
</div>

<script type="module">
    // استدعاء مكتبة PDF.js
    import * as pdfjsLib from "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.4.168/pdf.min.mjs"; 
    pdfjsLib.GlobalWorkerOptions.workerSrc = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.4.168/pdf.worker.min.mjs"; 

    const fileInput = document.getElementById("pdfFile"); 
    const textBox = document.getElementById("text"); 
    const status = document.getElementById("status"); 

    fileInput.addEventListener("change", async () => { 
        const file = fileInput.files[0]; 
        if (!file) return; 
        
        status.innerText = "⏳ جاري فتح الكتاب..."; 
        textBox.value = ""; // تفريغ النص القديم
        
        try { 
            const buffer = await file.arrayBuffer(); 
            const pdf = await pdfjsLib.getDocument({ data: buffer }).promise; 
            let completeText = ""; 
            
            for (let pageNo = 1; pageNo <= pdf.numPages; pageNo++) { 
                status.innerText = `📄 معالجة الصفحة ${pageNo} من ${pdf.numPages}`; 
                const page = await pdf.getPage(pageNo); 
                const content = await page.getTextContent(); 
                let pageText = content.items.map(x => x.str).join(" "); 
                
                /* إذا الصفحة تحتوي نص واضح، نستخدم النص مباشرة. */ 
                if (pageText.trim().length > 20) { 
                    completeText += pageText + "\n\n"; 
                } else { 
                    /* الصفحة غالباً صورة، لذلك نشغل OCR العربي. */ 
                    status.innerText = `🔎 الصفحة ${pageNo}: جاري التعرف على النص العربي عبر الـ OCR...`; 
                    const viewport = page.getViewport({ scale: 1.7 }); 
                    const canvas = document.createElement("canvas"); 
                    const context = canvas.getContext("2d"); 
                    canvas.width = viewport.width; 
                    canvas.height = viewport.height; 
                    
                    await page.render({ canvasContext: context, viewport: viewport }).promise; 
                    
                    const result = await Tesseract.recognize( 
                        canvas, 
                        "ara", 
                        { 
                            logger: data => { 
                                if (data.status === "recognizing text") { 
                                    const percent = Math.round(data.progress * 100); 
                                    status.innerText = `🔎 OCR عربي (الصفحة ${pageNo}): ${percent}%`; 
                                } 
                            } 
                        } 
                    ); 
                    completeText += result.data.text + "\n\n"; 
                } 
            } 
            
            textBox.value = completeText.trim(); 
            if (textBox.value.length > 0) { 
                status.innerText = "✅ تم تجهيز الكتاب بنجاح. اضغط «اقرأ»."; 
            } else { 
                status.innerText = "❌ لم أستطع استخراج النص من هذا الملف."; 
            } 
        } catch (error) { 
            console.error(error); 
            status.innerText = "❌ حدث خطأ أثناء معالجة ملف PDF."; 
        } 
    }); 

    /* ========================= القراءة الصوتية ========================= */ 
    let chunks = []; 
    let currentChunk = 0; 

    function splitText(text) { 
        // تقسيم النص إلى أجزاء صغيرة ليتحملها محرك النطق بالمتصفح دون مشاكل
        return text.match(/[\s\S]{1,1500}/g) || []; 
    } 

    window.readText = function() { 
        const text = textBox.value.trim(); 
        if (!text) { 
            alert("الرجاء استخراج أو كتابة نص أولاً قبل القراءة."); 
            return; 
        } 
        
        speechSynthesis.cancel(); // إيقاف أي قراءة سابقة
        chunks = splitText(text); 
        currentChunk = 0; 
        speakChunk(); 
    }; 

    function speakChunk() { 
        if (currentChunk >= chunks.length) { 
            status.innerText = "✅ انتهت القراءة بالكامل."; 
            return; 
        } 
        
        const utterance = new SpeechSynthesisUtterance(chunks[currentChunk]); 
        utterance.lang = "ar-SA"; // تعيين النطق باللهجة العربية السعودية كمثال قياسي
        utterance.rate = 0.85; 
        utterance.pitch = 1; 
        utterance.volume = 1; 
        
        utterance.onstart = function() { 
            status.innerText = `🔊 قراءة الجزء ${currentChunk + 1} من ${chunks.length}`; 
        }; 
        
        utterance.onend = function() { 
            currentChunk++; 
            speakChunk(); 
        }; 
        
        utterance.onerror = function(e) {
            console.error("خطأ في النطق الصوتي:", e);
            status.innerText = "❌ حدث خطأ أثناء النطق الصوتي.";
        };

        speechSynthesis.speak(utterance); 
    } 

    window.pauseText = function() { 
        if (speechSynthesis.speaking && !speechSynthesis.paused) {
            speechSynthesis.pause(); 
            status.innerText = "⏸️ تم التوقف مؤقتاً";
        }
    }; 

    window.resumeText = function() { 
        if (speechSynthesis.paused) {
            speechSynthesis.resume(); 
            status.innerText = `🔊 العودة لقراءة الجزء ${currentChunk + 1}`;
        }
    }; 

    window.stopText = function() { 
        speechSynthesis.cancel(); 
        currentChunk = 0;
        status.innerText = "⏹️ تم إيقاف القراءة بالكامل"; 
    };
</script>

</body>
</html>
