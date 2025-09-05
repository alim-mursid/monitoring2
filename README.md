<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Scalping Chart Analyzer</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #111827; /* Dark background */
        }
        .prose {
            color: #d1d5db; /* Lighter text for prose */
        }
        .prose h2 {
            color: #f9fafb;
        }
        .loader {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #3498db;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        /* Custom file input */
        .custom-file-input::-webkit-file-upload-button {
            visibility: hidden;
        }
        .custom-file-input::before {
            content: 'Pilih Gambar';
            display: inline-block;
            background: #3b82f6;
            color: white;
            border-radius: 0.375rem;
            padding: 0.5rem 1rem;
            outline: none;
            white-space: nowrap;
            -webkit-user-select: none;
            cursor: pointer;
            font-weight: 500;
        }
    </style>
</head>
<body class="text-gray-200">

    <div class="min-h-screen flex items-center justify-center p-4">
        <div class="w-full max-w-3xl bg-gray-900/70 backdrop-blur-sm border border-gray-700 rounded-2xl shadow-2xl p-6 md:p-10">
            
            <!-- Header -->
            <div class="text-center mb-8">
                <h1 class="text-3xl md:text-4xl font-bold text-white tracking-tight">AI Scalping Chart Analyzer</h1>
                <p class="mt-2 text-gray-400">Unggah screenshot chart trading Anda untuk mendapatkan analisis scalping dari AI.</p>
            </div>

            <!-- Image Upload Area -->
            <div class="mb-6">
                <label for="chartImage" class="block text-sm font-medium text-gray-300 mb-2">Upload Screenshot Chart:</label>
                <div class="mt-1 flex justify-center px-6 pt-5 pb-6 border-2 border-gray-600 border-dashed rounded-md" id="drop-zone">
                    <div class="space-y-1 text-center">
                        <svg class="mx-auto h-12 w-12 text-gray-500" stroke="currentColor" fill="none" viewBox="0 0 48 48" aria-hidden="true">
                            <path d="M28 8H12a4 4 0 00-4 4v20m32-12v8m0 0v8a4 4 0 01-4 4H12a4 4 0 01-4-4v-4m32-4l-3.172-3.172a4 4 0 00-5.656 0L28 28M8 32l9.172-9.172a4 4 0 015.656 0L28 28m0 0l4 4m4-24h8m-4-4v8" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                        </svg>
                        <div class="flex text-sm text-gray-500">
                            <label for="chartImage" class="relative cursor-pointer bg-gray-800 rounded-md font-medium text-blue-400 hover:text-blue-500 focus-within:outline-none focus-within:ring-2 focus-within:ring-offset-2 focus-within:ring-offset-gray-900 focus-within:ring-blue-500">
                                <span>Pilih file</span>
                                <input id="chartImage" name="chartImage" type="file" class="sr-only" accept="image/*">
                            </label>
                            <p class="pl-1">atau tarik dan letakkan di sini</p>
                        </div>
                        <p class="text-xs text-gray-600">PNG, JPG, GIF hingga 10MB</p>
                    </div>
                </div>
            </div>
            
            <!-- Image Preview -->
            <div id="imagePreview" class="mb-6 hidden">
                <p class="text-sm font-medium text-gray-300 mb-2">Preview:</p>
                <img id="preview" src="#" alt="Image Preview" class="rounded-lg max-h-96 w-auto mx-auto border-2 border-gray-700"/>
            </div>

            <!-- Analyze Button -->
            <div class="text-center">
                <button id="analyzeBtn" class="bg-blue-600 text-white font-semibold py-3 px-8 rounded-lg shadow-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-900 focus:ring-blue-500 transition-transform transform hover:scale-105 disabled:bg-gray-500 disabled:cursor-not-allowed disabled:scale-100">
                    Analisa Gambar
                </button>
            </div>

            <!-- Result Area -->
            <div id="resultContainer" class="mt-10 hidden">
                <h2 class="text-2xl font-bold text-white mb-4 border-b-2 border-gray-700 pb-2">Hasil Analisa AI</h2>
                <div id="loader" class="loader mx-auto my-8 hidden"></div>
                <div id="analysisResult" class="prose prose-invert max-w-none bg-gray-800/50 p-6 rounded-lg whitespace-pre-wrap"></div>
            </div>

        </div>
    </div>

    <script>
        // DOM Element References
        const chartImageInput = document.getElementById('chartImage');
        const analyzeBtn = document.getElementById('analyzeBtn');
        const imagePreviewContainer = document.getElementById('imagePreview');
        const previewImage = document.getElementById('preview');
        const resultContainer = document.getElementById('resultContainer');
        const loader = document.getElementById('loader');
        const analysisResult = document.getElementById('analysisResult');
        const dropZone = document.getElementById('drop-zone');

        // Gemini API Configuration
        const apiKey = ""; // Dibiarkan kosong, akan diinjeksi oleh environment
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;

        // Function to convert file to Base64
        function fileToBase64(file) {
            return new Promise((resolve, reject) => {
                const reader = new FileReader();
                reader.readAsDataURL(file);
                reader.onload = () => resolve(reader.result.split(',')[1]); // Hanya ambil data base64 nya
                reader.onerror = error => reject(error);
            });
        }
        
        // Handle file selection
        const handleFileSelect = (file) => {
            if (file && file.type.startsWith('image/')) {
                const reader = new FileReader();
                reader.onload = (e) => {
                    previewImage.src = e.target.result;
                    imagePreviewContainer.classList.remove('hidden');
                };
                reader.readAsDataURL(file);
            }
        };

        chartImageInput.addEventListener('change', (e) => {
            handleFileSelect(e.target.files[0]);
        });

        // Drag and Drop functionality
        dropZone.addEventListener('dragover', (e) => {
            e.preventDefault();
            dropZone.classList.add('border-blue-500', 'bg-gray-800');
        });

        dropZone.addEventListener('dragleave', (e) => {
            e.preventDefault();
            dropZone.classList.remove('border-blue-500', 'bg-gray-800');
        });

        dropZone.addEventListener('drop', (e) => {
            e.preventDefault();
            dropZone.classList.remove('border-blue-500', 'bg-gray-800');
            const file = e.dataTransfer.files[0];
            chartImageInput.files = e.dataTransfer.files; // Assign file to input
            handleFileSelect(file);
        });


        // Handle analysis button click
        analyzeBtn.addEventListener('click', async () => {
            const file = chartImageInput.files[0];
            if (!file) {
                analysisResult.textContent = 'Silakan pilih gambar terlebih dahulu.';
                resultContainer.classList.remove('hidden');
                return;
            }

            // --- UI State Update: Start Loading ---
            analyzeBtn.disabled = true;
            analyzeBtn.textContent = 'Menganalisa...';
            resultContainer.classList.remove('hidden');
            analysisResult.classList.add('hidden');
            loader.classList.remove('hidden');

            try {
                const base64Image = await fileToBase64(file);

                const systemPrompt = `Anda adalah seorang analis trading profesional yang sangat ahli dalam strategi scalping. Tugas Anda adalah menganalisis gambar grafik trading yang diberikan. Fokus secara eksklusif pada:
1.  **Identifikasi Pola Scalping:** Cari pola candlestick jangka pendek (seperti pin bar, engulfing, doji), formasi chart (seperti flag atau pennant kecil), atau sinyal dari indikator umum (jika terlihat) yang relevan untuk scalping (timeframe M1, M5, M15).
2.  **Potensi Entry Point:** Berikan saran titik masuk (entry) yang spesifik (level harga) untuk posisi Beli (Long) atau Jual (Short) berdasarkan pola yang Anda temukan.
3.  **Target Profit (TP):** Sarankan 1-2 level harga untuk Take Profit yang realistis untuk scalping (risk-reward ratio kecil, misal 1:1 atau 1:1.5).
4.  **Stop Loss (SL):** Sarankan level harga Stop Loss yang ketat untuk melindungi modal.
5.  **Ringkasan dan Keyakinan:** Berikan kesimpulan singkat tentang kualitas sinyal trading tersebut (misal: "Sinyal Beli dengan keyakinan sedang" atau "Pola kurang jelas, disarankan menunggu konfirmasi").

Gunakan format jawaban yang jelas dan terstruktur dengan poin-poin. Jangan memberikan nasihat keuangan. Analisis harus bersifat teknis dan objektif berdasarkan gambar.`;

                const payload = {
                    systemInstruction: {
                        parts: [{ text: systemPrompt }]
                    },
                    contents: [{
                        parts: [
                            { text: "Analisa gambar grafik trading ini untuk peluang scalping." },
                            {
                                inlineData: {
                                    mimeType: file.type,
                                    data: base64Image
                                }
                            }
                        ]
                    }],
                };

                // --- API Call ---
                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                if (!response.ok) {
                    throw new Error(`API Error: ${response.status} ${response.statusText}`);
                }

                const result = await response.json();
                
                // --- Process and Display Result ---
                if (result.candidates && result.candidates[0].content && result.candidates[0].content.parts[0].text) {
                    const analysisText = result.candidates[0].content.parts[0].text;
                    analysisResult.textContent = analysisText;
                } else {
                    analysisResult.textContent = 'Gagal mendapatkan analisis dari AI. Respon tidak valid. Coba lagi.';
                    console.error("Invalid API response structure:", result);
                }

            } catch (error) {
                console.error('Error:', error);
                analysisResult.textContent = `Terjadi kesalahan: ${error.message}. Silakan coba lagi.`;
            } finally {
                // --- UI State Update: End Loading ---
                analyzeBtn.disabled = false;
                analyzeBtn.textContent = 'Analisa Gambar';
                loader.classList.add('hidden');
                analysisResult.classList.remove('hidden');
            }
        });
    </script>

</body>
</html>
