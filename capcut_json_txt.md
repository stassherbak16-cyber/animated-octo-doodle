<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Экстрактор субтитров из CapCut (draft_info.json)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        .drag-active {
            border-color: #6366f1 !important;
            background-color: #e0e7ff !important;
        }
    </style>
</head>
<body class="bg-slate-900 text-slate-100 min-h-screen font-sans flex flex-col items-center justify-center p-4">

    <div class="max-w-3xl w-full bg-slate-800 border border-slate-700 rounded-2xl shadow-2xl p-6 sm:p-8 space-y-6">
        
        <!-- Header -->
        <div class="text-center space-y-2">
            <h1 class="text-2xl sm:text-3xl font-bold text-white flex items-center justify-center gap-2">
                <span>📝</span> Извлечение текста из CapCut
            </h1>
            <p class="text-slate-400 text-sm"> Загрузите файл <code class="bg-slate-700 px-2 py-0.5 rounded text-indigo-300">draft_info.json</code>, чтобы получить файл <code class="bg-slate-700 px-2 py-0.5 rounded text-emerald-300">.txt</code></p>
        </div>

        <!-- File Upload / Drag & Drop -->
        <div id="drop-zone" class="border-2 border-dashed border-slate-600 rounded-xl p-8 text-center cursor-pointer hover:border-indigo-500 hover:bg-slate-700/50 transition duration-200 flex flex-col items-center justify-center space-y-3">
            <svg class="w-12 h-12 text-slate-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"></path>
            </svg>
            <p class="text-sm sm:text-base font-medium text-slate-300">
                Перетащите сюда файл <span class="text-indigo-400">draft_info.json</span> или нажмите для выбора
            </p>
            <input type="file" id="file-input" accept=".json" class="hidden">
        </div>

        <!-- Paste option -->
        <details class="text-xs text-slate-400">
            <summary class="cursor-pointer hover:text-slate-200 select-none">Или вставьте содержимое JSON вручную</summary>
            <textarea id="raw-json-input" class="w-full mt-2 h-28 bg-slate-900 border border-slate-700 rounded-lg p-3 text-slate-300 font-mono text-xs focus:ring-2 focus:ring-indigo-500 focus:outline-none" placeholder="Вставьте JSON текст сюда..."></textarea>
            <button id="process-raw-btn" class="mt-2 px-3 py-1.5 bg-slate-700 hover:bg-slate-600 text-white rounded text-xs transition">Обработать вставленный текст</button>
        </details>

        <!-- Format Options -->
        <div class="flex flex-wrap items-center justify-between gap-4 border-t border-slate-700 pt-4">
            <label class="text-sm font-medium text-slate-300">Формат экспорта:</label>
            <div class="flex flex-wrap gap-2">
                <button data-format="plain" class="fmt-btn px-3 py-1.5 rounded-lg text-xs font-semibold bg-indigo-600 text-white shadow">Чистый текст</button>
                <button data-format="timestamps" class="fmt-btn px-3 py-1.5 rounded-lg text-xs font-semibold bg-slate-700 text-slate-300 hover:bg-slate-600">С таймкодами [00:12]</button>
                <button data-format="srt" class="fmt-btn px-3 py-1.5 rounded-lg text-xs font-semibold bg-slate-700 text-slate-300 hover:bg-slate-600">Формат субтитров (SRT)</button>
            </div>
        </div>

        <!-- Result Textarea -->
        <div class="space-y-2">
            <div class="flex justify-between items-center text-xs text-slate-400">
                <span id="status-text">Ожидание файла...</span>
                <span id="count-text"></span>
            </div>
            <textarea id="output-text" readonly class="w-full h-60 bg-slate-900 border border-slate-700 rounded-xl p-4 text-slate-200 font-mono text-sm focus:outline-none resize-y" placeholder="Результат появится здесь..."></textarea>
        </div>

        <!-- Action Buttons -->
        <div class="flex flex-col sm:flex-row gap-3 pt-2">
            <button id="download-btn" disabled class="flex-1 py-3 px-4 bg-emerald-600 hover:bg-emerald-500 disabled:bg-slate-700 disabled:opacity-50 text-white font-semibold rounded-xl transition duration-150 flex items-center justify-center gap-2">
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"></path></svg>
                Скачать .TXT
            </button>
            <button id="copy-btn" disabled class="py-3 px-4 bg-slate-700 hover:bg-slate-600 disabled:opacity-50 text-white font-semibold rounded-xl transition duration-150 flex items-center justify-center gap-2">
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"></path></svg>
                Скопировать
            </button>
        </div>

    </div>

    <script>
        let currentJSON = null;
        let selectedFormat = 'plain';

        const dropZone = document.getElementById('drop-zone');
        const fileInput = document.getElementById('file-input');
        const outputText = document.getElementById('output-text');
        const downloadBtn = document.getElementById('download-btn');
        const copyBtn = document.getElementById('copy-btn');
        const statusText = document.getElementById('status-text');
        const countText = document.getElementById('count-text');
        const rawJsonInput = document.getElementById('raw-json-input');
        const processRawBtn = document.getElementById('process-raw-btn');
        const fmtBtns = document.querySelectorAll('.fmt-btn');

        // Drag & Drop
        dropZone.addEventListener('click', () => fileInput.click());
        dropZone.addEventListener('dragover', (e) => { e.preventDefault(); dropZone.classList.add('drag-active'); });
        dropZone.addEventListener('dragleave', () => dropZone.classList.remove('drag-active'));
        dropZone.addEventListener('drop', (e) => {
            e.preventDefault();
            dropZone.classList.remove('drag-active');
            if (e.dataTransfer.files.length) handleFile(e.dataTransfer.files[0]);
        });

        fileInput.addEventListener('change', (e) => {
            if (e.target.files.length) handleFile(e.target.files[0]);
        });

        processRawBtn.addEventListener('click', () => {
            const val = rawJsonInput.value.trim();
            if (!val) return;
            try {
                currentJSON = JSON.parse(val);
                processJSON();
            } catch (err) {
                alert('Некорректный формат JSON!');
            }
        });

        // Format Switchers
        fmtBtns.forEach(btn => {
            btn.addEventListener('click', () => {
                fmtBtns.forEach(b => {
                    b.classList.remove('bg-indigo-600', 'text-white');
                    b.classList.add('bg-slate-700', 'text-slate-300');
                });
                btn.classList.remove('bg-slate-700', 'text-slate-300');
                btn.classList.add('bg-indigo-600', 'text-white');
                selectedFormat = btn.dataset.format;
                if (currentJSON) processJSON();
            });
        });

        function handleFile(file) {
            const reader = new FileReader();
            reader.onload = (e) => {
                try {
                    currentJSON = JSON.parse(e.target.result);
                    statusText.textContent = `Загружен файл: ${file.name}`;
                    processJSON();
                } catch (err) {
                    alert('Ошибка при чтении JSON файла. Убедитесь, что это правильный draft_info.json');
                }
            };
            reader.readAsText(file);
        }

        function cleanText(str) {
            if (!str) return '';
            // Handle if content is internal JSON
            try {
                const parsed = JSON.parse(str);
                if (parsed.text) str = parsed.text;
            } catch(e) {}
            
            // Remove xml tags like <font...>, <color...>, etc.
            return str.replace(/<[^>]*>/g, '').trim();
        }

        function usToSrtTime(us) {
            let totalMs = Math.floor(us / 1000);
            let ms = (totalMs % 1000).toString().padStart(3, '0');
            let totalSec = Math.floor(totalMs / 1000);
            let s = (totalSec % 60).toString().padStart(2, '0');
            let totalMin = Math.floor(totalSec / 60);
            let m = (totalMin % 60).toString().padStart(2, '0');
            let h = Math.floor(totalMin / 60).toString().padStart(2, '0');
            return `${h}:${m}:${s},${ms}`;
        }

        function usToSimpleTime(us) {
            let totalSec = Math.floor(us / 1000000);
            let m = Math.floor(totalSec / 60).toString().padStart(2, '0');
            let s = (totalSec % 60).toString().padStart(2, '0');
            return `${m}:${s}`;
        }

        function processJSON() {
            if (!currentJSON) return;

            // 1. Map text materials
            const textMap = {};
            if (currentJSON.materials && currentJSON.materials.texts) {
                currentJSON.materials.texts.forEach(item => {
                    let content = cleanText(item.content || item.text || '');
                    if (content) {
                        textMap[item.id] = content;
                    }
                });
            }

            // 2. Extract segments from tracks
            let segments = [];
            if (currentJSON.tracks) {
                currentJSON.tracks.forEach(track => {
                    if (track.type === 'text' || track.type === 2) {
                        (track.segments || []).forEach(seg => {
                            let text = textMap[seg.material_id] || cleanText(seg.name);
                            let start = seg.target_timerange ? (seg.target_timerange.start || 0) : 0;
                            let duration = seg.target_timerange ? (seg.target_timerange.duration || 0) : 0;

                            if (text) {
                                segments.push({
                                    start: Number(start),
                                    duration: Number(duration),
                                    end: Number(start) + Number(duration),
                                    text: text
                                });
                            }
                        });
                    }
                });
            }

            // Fallback if no tracks matched directly
            if (segments.length === 0 && Object.keys(textMap).length > 0) {
                let idx = 0;
                for (let id in textMap) {
                    segments.push({
                        start: idx * 2000000,
                        duration: 2000000,
                        end: (idx + 1) * 2000000,
                        text: textMap[id]
                    });
                    idx++;
                }
            }

            // Sort chronologically
            segments.sort((a, b) => a.start - b.start);

            countText.textContent = `Найдено строк: ${segments.length}`;

            if (segments.length === 0) {
                outputText.value = "В файле не найдено текстовых субтитров или надписей.";
                downloadBtn.disabled = true;
                copyBtn.disabled = true;
                return;
            }

            // Format result
            let result = '';
            if (selectedFormat === 'plain') {
                result = segments.map(s => s.text).join('\n');
            } else if (selectedFormat === 'timestamps') {
                result = segments.map(s => `[${usToSimpleTime(s.start)}] ${s.text}`).join('\n');
            } else if (selectedFormat === 'srt') {
                result = segments.map((s, i) => `${i + 1}\n${usToSrtTime(s.start)} --> ${usToSrtTime(s.end)}\n${s.text}\n`).join('\n');
            }

            outputText.value = result;
            downloadBtn.disabled = false;
            copyBtn.disabled = false;
        }

        // Download Action
        downloadBtn.addEventListener('click', () => {
            const content = outputText.value;
            if (!content) return;
            
            const ext = selectedFormat === 'srt' ? 'srt' : 'txt';
            const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `capcut_subtitles.${ext}`;
            a.click();
            URL.revokeObjectURL(url);
        });

        // Copy Action
        copyBtn.addEventListener('click', () => {
            outputText.select();
            navigator.clipboard.writeText(outputText.value);
            const originalText = copyBtn.innerText;
            copyBtn.innerText = 'Скопировано!';
            setTimeout(() => copyBtn.innerText = originalText, 2000);
        });
    </script>
</body>
</html>