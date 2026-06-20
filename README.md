<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PDF 편집기</title>
    <script src="https://cdn.jsdelivr.net/npm/pdf-lib@1.17.1/dist/pdf-lib.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;500;600;700&display=swap');
        
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Noto Sans KR', sans-serif;
            background: linear-gradient(135deg, #f1f5f9, #dbeafe);
            color: #1e2937;
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 25px -5px rgba(0,0,0,0.1);
            overflow: hidden;
        }
        header {
            text-align: center;
            padding: 40px 20px;
            background: linear-gradient(135deg, #2563eb, #1e40af);
            color: white;
        }
        h1 { font-size: 2.5rem; font-weight: 700; }

        .section { padding: 35px; }
        
        .upload-area {
            border: 3px dashed #94a3b8;
            border-radius: 16px;
            padding: 70px 30px;
            text-align: center;
            cursor: pointer;
            transition: all 0.3s;
        }
        .upload-area:hover, .upload-area.dragover {
            border-color: #2563eb;
            background: #eff6ff;
            transform: scale(1.02);
        }
        
        .preview-container {
            margin-top: 30px;
            border: 1px solid #e2e8f0;
            border-radius: 16px;
            background: #f8fafc;
            padding: 20px;
        }
        .pages-list {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(190px, 1fr));
            gap: 20px;
        }
        .page-item {
            background: white;
            border: 2px solid #e2e8f0;
            border-radius: 14px;
            padding: 12px;
            position: relative;
            cursor: move;
            transition: all 0.25s ease;
        }
        .page-item:hover { 
            box-shadow: 0 15px 20px -5px rgba(0,0,0,0.12);
            transform: translateY(-4px);
        }
        .page-item.selected { border-color: #2563eb; box-shadow: 0 0 0 4px rgba(37,99,235,0.2); }

        .page-thumb {
            width: 100%;
            height: 230px;
            background: #f8fafc;
            border-radius: 10px;
            overflow: hidden;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .page-thumb img {
            width: 100%;
            height: 100%;
            object-fit: contain;
        }

        .delete-btn {
            position: absolute;
            top: -10px;
            right: -10px;
            background: #ef4444;
            color: white;
            border: 3px solid white;
            width: 36px;
            height: 36px;
            border-radius: 50%;
            font-size: 20px;
            font-weight: 700;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            box-shadow: 0 4px 10px rgba(239, 68, 68, 0.4);
            z-index: 20;
            transition: all 0.2s;
        }
        .delete-btn:hover {
            background: #dc2626;
            transform: scale(1.1);
        }

        .page-number {
            position: absolute;
            bottom: 12px;
            right: 12px;
            background: rgba(0,0,0,0.8);
            color: white;
            font-size: 0.85rem;
            padding: 4px 11px;
            border-radius: 9999px;
            font-weight: 500;
        }

        .toolbar { text-align: center; margin: 35px 0; }
        button {
            padding: 14px 34px;
            margin: 0 8px;
            border: none;
            border-radius: 12px;
            font-size: 1.08rem;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.25s;
        }
        .btn-primary {
            background: #2563eb;
            color: white;
        }
        .btn-primary:hover { background: #1d4ed8; transform: translateY(-3px); }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>📄 PDF 편집기</h1>
            <p>페이지 삭제 · 순서 변경 · 새로운 PDF 저장</p>
        </header>
        
        <div class="section">
            <div class="upload-area" id="pdf-upload-area">
                <h3>PDF 파일을 여기에 끌어다 놓으세요</h3>
                <p style="color:#64748b; margin-top:8px;">여러 파일 동시 업로드 가능</p>
                <button class="btn-primary" onclick="document.getElementById('pdf-input').click()" style="margin-top:20px;">
                    파일 선택
                </button>
                <input type="file" id="pdf-input" accept=".pdf" multiple style="display:none;">
            </div>

            <div id="pdf-preview" class="preview-container" style="display:none;">
                <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:20px;">
                    <h3>페이지 목록 <span id="page-count"></span></h3>
                    <div>
                        <button onclick="deleteSelectedPages()" class="btn-danger" style="background:#ef4444;">선택 삭제</button>
                        <button onclick="clearAllPages()" class="btn-danger" style="background:#ef4444;">전체 초기화</button>
                    </div>
                </div>
                <div id="pages-list" class="pages-list"></div>
            </div>

            <div class="toolbar">
                <button onclick="downloadPDF()" class="btn-primary" style="padding:18px 60px; font-size:1.15rem;">
                    📥 새로운 PDF 다운로드
                </button>
            </div>
        </div>
    </div>

    <script>
        pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
        
        let allPages = [];

        const uploadArea = document.getElementById('pdf-upload-area');
        const pdfInput = document.getElementById('pdf-input');

        uploadArea.addEventListener('dragover', e => { e.preventDefault(); uploadArea.classList.add('dragover'); });
        uploadArea.addEventListener('dragleave', () => uploadArea.classList.remove('dragover'));
        uploadArea.addEventListener('drop', e => { e.preventDefault(); uploadArea.classList.remove('dragover'); handleFiles(e.dataTransfer.files); });

        pdfInput.addEventListener('change', e => e.target.files.length && handleFiles(e.target.files));

        async function handleFiles(files) {
            for (let file of files) {
                if (!file.name.toLowerCase().endsWith('.pdf')) continue;
                const arrayBuffer = await file.arrayBuffer();
                const pdfDoc = await PDFLib.PDFDocument.load(arrayBuffer);
                const pdfJsDoc = await pdfjsLib.getDocument({data: arrayBuffer}).promise;

                for (let i = 0; i < pdfDoc.getPageCount(); i++) {
                    const page = await pdfJsDoc.getPage(i + 1);
                    const viewport = page.getViewport({scale: 0.85});
                    const canvas = document.createElement('canvas');
                    canvas.height = viewport.height;
                    canvas.width = viewport.width;
                    await page.render({canvasContext: canvas.getContext('2d'), viewport}).promise;

                    allPages.push({
                        pdfDoc, pageIndex: i, name: file.name,
                        thumbUrl: canvas.toDataURL('image/jpeg', 0.9)
                    });
                }
            }
            renderPages();
        }

        function renderPages() {
            const container = document.getElementById('pdf-preview');
            const list = document.getElementById('pages-list');
            list.innerHTML = '';
            container.style.display = 'block';
            document.getElementById('page-count').textContent = `(${allPages.length}페이지)`;

            allPages.forEach((pageData, idx) => {
                const div = document.createElement('div');
                div.className = 'page-item';
                div.draggable = true;
                div.dataset.index = idx;

                div.innerHTML = `
                    <button class="delete-btn" onclick="deletePage(${idx}); event.stopImmediatePropagation();">×</button>
                    <div class="page-thumb"><img src="${pageData.thumbUrl}" alt="페이지"></div>
                    <div class="page-number">${idx + 1}</div>
                    <div style="text-align:center; margin-top:10px; font-size:0.9rem; color:#64748b;">
                        ${pageData.name}<br><small>페이지 ${pageData.pageIndex + 1}</small>
                    </div>
                `;

                // Drag & Drop 개선
                div.addEventListener('dragstart', e => {
                    e.dataTransfer.setData('text/plain', idx);
                    div.style.opacity = '0.4';
                    div.style.transform = 'scale(0.95)';
                });
                div.addEventListener('dragend', () => {
                    div.style.opacity = '1';
                    div.style.transform = 'scale(1)';
                });
                div.addEventListener('dragover', e => e.preventDefault());
                div.addEventListener('drop', e => {
                    e.preventDefault();
                    const from = parseInt(e.dataTransfer.getData('text/plain'));
                    const to = parseInt(div.dataset.index);
                    if (from !== to) {
                        const [moved] = allPages.splice(from, 1);
                        allPages.splice(to, 0, moved);
                        renderPages();
                    }
                });

                div.addEventListener('click', () => div.classList.toggle('selected'));
                list.appendChild(div);
            });
        }

        function deletePage(idx) {
            allPages.splice(idx, 1);
            renderPages();
        }

        function deleteSelectedPages() {
            const selected = [...document.querySelectorAll('.page-item.selected')]
                .map(el => parseInt(el.dataset.index)).sort((a,b)=>b-a);
            selected.forEach(i => allPages.splice(i, 1));
            renderPages();
        }

        function clearAllPages() {
            if (confirm('모든 페이지를 초기화하시겠습니까?')) {
                allPages = [];
                document.getElementById('pdf-preview').style.display = 'none';
            }
        }

        async function downloadPDF() {
            if (allPages.length === 0) return alert('편집할 페이지가 없습니다.');
            const newPdf = await PDFLib.PDFDocument.create();
            for (let p of allPages) {
                const [page] = await newPdf.copyPages(p.pdfDoc, [p.pageIndex]);
                newPdf.addPage(page);
            }
            const bytes = await newPdf.save();
            const blob = new Blob([bytes], {type: 'application/pdf'});
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url; a.download = `편집된_PDF_${new Date().toISOString().slice(0,10)}.pdf`;
            a.click();
            URL.revokeObjectURL(url);
        }
    </script>
</body>
</html>
