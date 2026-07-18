<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Media Downloader</title>
    <style>
        :root {
            --primary: #007BFF;
            --primary-hover: #0056b3;
            --bg-color: #f8f9fa;
            --text-color: #333;
            --border-radius: 8px;
        }
        
        body { 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            max-width: 600px; 
            margin: 50px auto; 
            padding: 20px;
            text-align: center; 
        }
        
        .container {
            background: white;
            padding: 30px;
            border-radius: var(--border-radius);
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }

        h1 { margin-top: 0; color: #222; }
        
        .input-group {
            display: flex;
            flex-direction: column;
            gap: 15px;
            margin-bottom: 20px;
        }

        input, select { 
            padding: 12px; 
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 4px;
            width: 100%;
            box-sizing: border-box;
        }
        
        button { 
            padding: 12px 20px; 
            font-size: 16px; 
            font-weight: bold;
            cursor: pointer; 
            background: var(--primary); 
            color: white; 
            border: none; 
            border-radius: 4px;
            transition: background 0.2s;
            width: 100%;
        }
        
        button:hover { background: var(--primary-hover); }
        button:disabled { background: #ccc; cursor: not-allowed; }
        
        #status { 
            margin-top: 20px; 
            padding: 10px;
            border-radius: 4px;
            display: none; /* Hidden until needed */
        }

        .status-processing { background: #e2e3e5; color: #383d41; display: block !important; }
        .status-success { background: #d4edda; color: #155724; display: block !important; }
        .status-error { background: #f8d7da; color: #721c24; display: block !important; }
        
        .loader {
            display: none;
            border: 3px solid #f3f3f3;
            border-top: 3px solid var(--primary);
            border-radius: 50%;
            width: 20px;
            height: 20px;
            animation: spin 1s linear infinite;
            margin: 0 auto 10px auto;
        }

        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Media Downloader</h1>
        <p>Extract video or audio files directly to your device.</p>
        
        <div class="input-group">
            <input type="url" id="videoUrl" placeholder="Paste YouTube URL here..." required>
            <select id="formatSelect">
                <option value="video">High Quality Video (MP4)</option>
                <option value="audio">Extract Audio Only (MP3)</option>
            </select>
        </div>

        <button id="downloadBtn" onclick="processDownload()">Download</button>
        
        <div id="status">
            <div class="loader" id="spinner"></div>
            <span id="statusText"></span>
        </div>
    </div>

    <script>
        async function processDownload() {
            const urlInput = document.getElementById('videoUrl');
            const url = urlInput.value.trim();
            const format = document.getElementById('formatSelect').value;
            const statusBox = document.getElementById('status');
            const statusText = document.getElementById('statusText');
            const spinner = document.getElementById('spinner');
            const btn = document.getElementById('downloadBtn');

            // Basic validation
            if (!url) {
                setStatus('error', "Please paste a valid URL first.");
                urlInput.focus();
                return;
            }

            // Set UI to loading state
            setStatus('processing', `Processing ${format === 'audio' ? 'Audio' : 'Video'}... Please wait.`);
            btn.disabled = true;
            spinner.style.display = 'block';

            try {
                // Send request to the Flask backend deployed on Render
                const response = await fetch('/download', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ url, format }) 
                });
                
                if (response.ok) {
                    // Convert the response stream into a downloadable file
                    const blob = await response.blob();
                    const downloadUrl = window.URL.createObjectURL(blob);
                    const a = document.createElement('a');
                    a.href = downloadUrl;
                    
                    // Attempt to extract the correct filename from the server response headers
                    const disposition = response.headers.get('Content-Disposition');
                    let filename = format === 'audio' ? 'extracted_audio.mp3' : 'downloaded_video.mp4'; 
                    
                    if (disposition && disposition.indexOf('attachment') !== -1) {
                        const filenameRegex = /filename[^;=\n]*=((['"]).*?\2|[^;\n]*)/;
                        const matches = filenameRegex.exec(disposition);
                        if (matches != null && matches[1]) { 
                            filename = matches[1].replace(/['"]/g, '');
                        }
                    }
                    
                    // Trigger the browser download
                    a.download = filename;
                    document.body.appendChild(a);
                    a.click();
                    a.remove();
                    
                    // Clean up object URL to free memory
                    setTimeout(() => window.URL.revokeObjectURL(downloadUrl), 100);
                    
                    setStatus('success', "Download complete!");
                } else {
                    // Handle errors returned by the Flask backend
                    const err = await response.json();
                    setStatus('error', err.error || "Download failed. The server might have timed out.");
                }
            } catch (error) {
                // Handle network errors (e.g., server offline, CORS issues)
                console.error("Fetch error:", error);
                setStatus('error', "Network connection error. Please try again later.");
            } finally {
                // Always re-enable the button and hide the spinner when done
                btn.disabled = false;
                spinner.style.display = 'none';
            }
        }

        // Helper function to manage status box styling
        function setStatus(type, message) {
            const statusBox = document.getElementById('status');
            const statusText = document.getElementById('statusText');
            
            // Remove old classes
            statusBox.className = '';
            
            // Apply new class and message
            statusBox.classList.add(`status-${type}`);
            statusText.innerText = message;
        }
    </script>
</body>
</html>

