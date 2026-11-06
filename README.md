<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cryptogram UK</title>
    <!-- CryptoJS for client-side AES encryption -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.1.1/crypto-js.min.js"></script>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; padding: 20px; max-width: 500px; margin: 0 auto; }
        .form-group { margin-bottom: 15px; }
        label { display: block; font-weight: bold; margin-bottom: 5px; font-size: 0.9rem; }
        input, button { width: 100%; padding: 10px; box-sizing: border-box; font-size: 1rem; border: 1px solid #ccc; border-radius: 4px; }
        button { background-color: #0056b3; color: white; border: none; font-weight: bold; cursor: pointer; margin-top: 10px; }
        button:hover { background-color: #004085; }
        pre { background: #f4f4f4; padding: 10px; border-radius: 4px; overflow-x: auto; }
        .status { margin-top: 10px; font-weight: bold; }
    </style>
</head>
<body>

    <h2>Cryptogram UK</h2>

    <div class="form-group">
        <label>GitHub Personal Access Token (PAT):</label>
        <input type="password" id="ghToken" value="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">
    </div>

    <div class="form-group">
        <label>Repository (username/repo):</label>
        <input type="text" id="ghRepo" value="mediaolit-hue/cryptogram">
    </div>

    <div class="form-group">
        <label>Encryption Passphrase:</label>
        <input type="password" id="passphrase" value="blackbird">
    </div>

    <hr>

    <div class="form-group">
        <label>Add Input (Format: Key Value, e.g. "L 10"):</label>
        <input type="text" id="tallyInput" placeholder="L 10">
    </div>

    <button onclick="addAndSync()">Add & Update GitHub</button>
    <button onclick="fetchAndDecrypt()" style="background-color: #333;">Fetch & Decrypt</button>

    <div class="status" id="statusText"></div>

    <h3>Current Tally (Decrypted):</h3>
    <pre id="tallyDisplay">{}</pre>

    <script>
        const FILE_PATH = "tally_data.enc";
        let localTally = {};
        let currentSha = ""; // Required by GitHub API for updating existing files

        function logStatus(msg) {
            document.getElementById('statusText').innerText = msg;
        }

        // --- ENCRYPTION HELPERS ---
        function encryptData(dataObject, secretKey) {
            const jsonString = JSON.stringify(dataObject);
            return CryptoJS.AES.encrypt(jsonString, secretKey).toString();
        }

        function decryptData(ciphertext, secretKey) {
            try {
                const bytes = CryptoJS.AES.decrypt(ciphertext, secretKey);
                const decryptedString = bytes.toString(CryptoJS.enc.Utf8);
                return decryptedString ? JSON.parse(decryptedString) : null;
            } catch (e) {
                return null;
            }
        }

        // --- GITHUB API OPERATIONS ---
        async function fetchAndDecrypt() {
            const token = document.getElementById('ghToken').value.trim();
            const repo = document.getElementById('ghRepo').value.trim();
            const pass = document.getElementById('passphrase').value.trim();

            if (!token || !repo || !pass) {
                alert("Please provide Token, Repo, and Passphrase.");
                return;
            }

            logStatus("Fetching from GitHub...");
            const url = `https://api.github.com/repos/${repo}/contents/${FILE_PATH}`;

            try {
                const response = await fetch(url, {
                    headers: { "Authorization": `token ${token}` }
                });

                if (response.status === 404) {
                    logStatus("File not found on GitHub. Starting new database.");
                    localTally = {};
                    currentSha = "";
                    document.getElementById('tallyDisplay').innerText = JSON.stringify(localTally, null, 2);
                    return;
                }

                const fileData = await response.json();
                currentSha = fileData.sha;
                
                // Decode base64 file contents from GitHub
                const encryptedContent = atob(fileData.content);
                const decrypted = decryptData(encryptedContent, pass);

                if (decrypted) {
                    localTally = decrypted;
                    document.getElementById('tallyDisplay').innerText = JSON.stringify(localTally, null, 2);
                    logStatus("Successfully decrypted data.");
                } else {
                    logStatus("Failed to decrypt! Incorrect passphrase.");
                }
            } catch (err) {
                logStatus("API Error: " + err.message);
            }
        }

        async function addAndSync() {
            const input = document.getElementById('tallyInput').value.trim();
            const pass = document.getElementById('passphrase').value.trim();
            const token = document.getElementById('ghToken').value.trim();
            const repo = document.getElementById('ghRepo').value.trim();

            if (!input) {
                alert("Please enter an entry like 'L 10'.");
                return;
            }

            // Parse input: "L 10" -> Key: L, Value: 10
            const parts = input.split(/\s+/);
            const key = parts[0];
            const val = parseFloat(parts[1]);

            if (!key || isNaN(val)) {
                alert("Invalid format. Use format: KEY VALUE (e.g. L 10)");
                return;
            }

            // 1. Update running local tally
            localTally[key] = (localTally[key] || 0) + val;
            document.getElementById('tallyDisplay').innerText = JSON.stringify(localTally, null, 2);

            // 2. Encrypt full updated database
            const encryptedPayload = encryptData(localTally, pass);
            const encodedContent = btoa(encryptedPayload);

            // 3. Commit to GitHub API
            logStatus("Syncing to GitHub...");
            const url = `https://api.github.com/repos/${repo}/contents/${FILE_PATH}`;
            
            const bodyData = {
                message: `Update tally: ${new Date().toISOString()}`,
                content: encodedContent
            };
            if (currentSha) {
                bodyData.sha = currentSha;
            }

            try {
                const res = await fetch(url, {
                    method: "PUT",
                    headers: {
                        "Authorization": `token ${token}`,
                        "Content-Type": "application/json"
                    },
                    body: JSON.stringify(bodyData)
                });

                if (res.ok) {
                    const result = await res.json();
                    currentSha = result.content.sha;
                    logStatus("Successfully saved encrypted data to GitHub!");
                    document.getElementById('tallyInput').value = '';
                } else {
                    const errRes = await res.json();
                    logStatus("Failed to save: " + errRes.message);
                }
            } catch (err) {
                logStatus("Network Error: " + err.message);
            }
        }
    </script>
</body>
</html>
