This is an ambitious and well-structured project\! Automating the generation of feature-building prompts using AI and managing them locally with a user-friendly interface is a great idea.

Here's a detailed breakdown of how you can achieve this using Python, covering the frontend, backend logic, API integrations, file handling, and best practices for separation of concerns and security.

### Project Structure (Separation of Concerns)

A good project structure is crucial for maintainability. I recommend the following:

```
feature_prompt_generator/
├─── frontend/
│ ├─── public/
│ │ └─── index.html
│ └─── src/
│ ├─── App.js
│ ├─── index.js
│ └─── styles.css
├─── backend/
│ ├─── app.py
│ ├─── api_clients/
│ │ ├─── __init__.py
│ │ ├─── openai_client.py
│ │ ├─── gemini_client.py
│ │ └─── anthropic_client.py
│ ├─── prompt_generator.py
│ ├─── file_handler.py
│ └─── config.py
├─── .env
├─── requirements.txt
└─── README.md
```

**Explanation:**

  * **`feature_prompt_generator/`**: The root directory of your project.
  * **`frontend/`**: Contains all frontend (React/HTML/JS/CSS) code.
      * `public/index.html`: The main HTML file.
      * `src/`: Your React source code.
  * **`backend/`**: Contains all Python backend logic.
      * `app.py`: The main Flask/FastAPI application, handles routes and connects frontend to backend logic.
      * `api_clients/`: Directory for LLM API client wrappers. Each file handles the specifics of one API (OpenAI, Gemini, Anthropic).
      * `prompt_generator.py`: Contains the core logic for generating prompts, utilizing the `api_clients`.
      * `file_handler.py`: Manages creating `.md` files and zipping them.
      * `config.py`: For application-wide configuration (though sensitive keys go in `.env`).
  * **`.env`**: Stores environment variables, especially API keys (crucial for security\!).
  * **`requirements.txt`**: Lists all Python dependencies.
  * **`README.md`**: Project documentation.

### Technologies to Use

  * **Backend (Python):**
      * **Flask or FastAPI:** For building the local web server and API endpoints. Flask is simpler for small projects, FastAPI is more modern and performant. Let's go with Flask for simplicity.
      * **`python-dotenv`:** For loading environment variables from `.env`.
      * **OpenAI Python Client:** For interacting with OpenAI models.
      * **Google Generative AI Python Client (`google-generative-ai` or `openai` library with Google's compatible base URL):** For interacting with Gemini models.
      * **Anthropic Python Client (`anthropic`):** For interacting with Anthropic models.
      * **`zipfile` (built-in):** For creating `.zip` archives.
      * **`os` (built-in):** For file system operations (creating directories, joining paths).
  * **Frontend (JavaScript/HTML/CSS):**
      * **React (recommended):** For a reactive and component-based UI. You can use `create-react-app` to quickly set up a basic React project.
      * **Axios or `fetch` API:** For making HTTP requests to your Python backend.

### Step-by-Step Implementation Guide

#### 1\. Backend Setup (`backend/`)

**1.1. `requirements.txt`**

```
Flask
python-dotenv
openai
google-generativeai # Or if using openai client for gemini: just openai
anthropic
```

**1.2. `.env` (Create this file in the root `feature_prompt_generator/` directory)**

```
OPENAI_API_KEY=your_openai_api_key_here
GEMINI_API_KEY=your_gemini_api_key_here
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

**Important:** Never commit `.env` to version control (add it to `.gitignore`)\!

**1.3. `backend/config.py`**

```python
import os

# Define the folder where generated prompts will be saved
OUTPUT_FOLDER = "generated_prompts"
ZIP_FILE_NAME = "generated_prompts.zip"
```

**1.4. `backend/api_clients/openai_client.py`**

```python
import os
from openai import OpenAI

class OpenAIClient:
    def __init__(self):
        self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

    def generate_prompt(self, system_prompt: str, user_prompt: str, model: str = "gpt-4o"):
        try:
            response = self.client.chat.completions.create(
                model=model,
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": user_prompt}
                ],
                max_tokens=1000,
                temperature=0.7,
            )
            return response.choices[0].message.content
        except Exception as e:
            print(f"Error calling OpenAI API: {e}")
            return None

```

**1.5. `backend/api_clients/gemini_client.py`**

```python
import os
import google.generativeai as genai

class GeminiClient:
    def __init__(self):
        genai.configure(api_key=os.getenv("GEMINI_API_KEY"))

    def generate_prompt(self, system_prompt: str, user_prompt: str, model: str = "gemini-pro"):
        try:
            model_instance = genai.GenerativeModel(model)
            response = model_instance.generate_content(
                contents=[
                    {"role": "user", "parts": [system_prompt]},
                    {"role": "user", "parts": [user_prompt]}
                ]
            )
            return response.text
        except Exception as e:
            print(f"Error calling Gemini API: {e}")
            return None

```

**1.6. `backend/api_clients/anthropic_client.py`**

```python
import os
import anthropic

class AnthropicClient:
    def __init__(self):
        self.client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

    def generate_prompt(self, system_prompt: str, user_prompt: str, model: str = "claude-3-opus-20240229"):
        try:
            response = self.client.messages.create(
                model=model,
                max_tokens=1000,
                messages=[
                    {"role": "user", "content": f"{system_prompt}\n{user_prompt}"}
                ],
                temperature=0.7,
            )
            return response.content[0].text
        except Exception as e:
            print(f"Error calling Anthropic API: {e}")
            return None

```

**1.7. `backend/prompt_generator.py`**

```python
from backend.api_clients.openai_client import OpenAIClient
from backend.api_clients.gemini_client import GeminiClient
from backend.api_clients.anthropic_client import AnthropicClient

class PromptGenerator:
    def __init__(self, api_provider: str):
        self.api_provider = api_provider
        self.client = self._get_client()

    def _get_client(self):
        if self.api_provider == "openai":
            return OpenAIClient()
        elif self.api_provider == "gemini":
            return GeminiClient()
        elif self.api_provider == "anthropic":
            return AnthropicClient()
        else:
            raise ValueError("Invalid API provider specified.")

    def generate_feature_prompt(self, filename: str, sample_prompt_text: str) -> str:
        system_prompt = f"You are an expert prompt engineer for a coding agent. Your goal is to generate a highly detailed and actionable prompt for a coding agent to build a specific software feature. The coding agent will read this prompt and generate code. Focus on clarity, completeness, and edge cases. Ensure the prompt is self-contained and provides all necessary context."
        
        user_prompt = f"""
I need a feature-building prompt for a coding agent. The feature should be saved in a file named: `{filename}`.

Here is a sample of the kind of prompt content I'm looking for, which you should adapt and expand significantly for the specific filename:

---
Sample Prompt Text:
{sample_prompt_text}
---

Please generate the complete, detailed feature-building prompt for `{filename}`.
It should include:
1.  **Feature Description:** A clear and concise description of the feature.
2.  **Requirements:** Detailed functional and non-functional requirements.
3.  **Input/Output (if applicable):** Expected inputs and desired outputs.
4.  **Edge Cases/Error Handling:** How the feature should behave in unusual or error conditions.
5.  **Technical Considerations (if any):** Specific technologies, libraries, or architectural patterns to consider.
6.  **Example Usage (if applicable):** A simple example of how the feature would be used.

The prompt should be written in Markdown format, ready to be saved as a .md file.
"""
        print(f"Generating prompt for {filename} using {self.api_provider}...")
        generated_content = self.client.generate_prompt(system_prompt, user_prompt)
        return generated_content

```

**1.8. `backend/file_handler.py`**

```python
import os
import zipfile
from backend.config import OUTPUT_FOLDER, ZIP_FILE_NAME

class FileHandler:
    def __init__(self):
        os.makedirs(OUTPUT_FOLDER, exist_ok=True)

    def save_markdown_file(self, filename: str, content: str):
        file_path = os.path.join(OUTPUT_FOLDER, filename)
        try:
            with open(file_path, "w", encoding="utf-8") as f:
                f.write(content)
            print(f"Saved: {file_path}")
            return True
        except Exception as e:
            print(f"Error saving file {file_path}: {e}")
            return False

    def create_zip_archive(self):
        zip_path = os.path.join(os.getcwd(), ZIP_FILE_NAME) # Save zip in root dir
        try:
            with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
                for root, _, files in os.walk(OUTPUT_FOLDER):
                    for file in files:
                        file_path = os.path.join(root, file)
                        # Add file to zip with a relative path within the zip
                        zipf.write(file_path, os.path.relpath(file_path, OUTPUT_FOLDER))
            print(f"Created zip archive: {zip_path}")
            return zip_path
        except Exception as e:
            print(f"Error creating zip archive: {e}")
            return None

    def clean_output_folder(self):
        for root, _, files in os.walk(OUTPUT_FOLDER):
            for file in files:
                os.remove(os.path.join(root, file))
        os.rmdir(OUTPUT_FOLDER)
        print(f"Cleaned output folder: {OUTPUT_FOLDER}")
```

**1.9. `backend/app.py` (Flask Application)**

```python
from flask import Flask, request, jsonify, send_file
from flask_cors import CORS # Needed for local development with separate frontend
from dotenv import load_dotenv
import os
import time

from backend.prompt_generator import PromptGenerator
from backend.file_handler import FileHandler
from backend.config import OUTPUT_FOLDER, ZIP_FILE_NAME

load_dotenv() # Load environment variables from .env

app = Flask(__name__)
CORS(app) # Enable CORS for all routes (for development)

@app.route('/generate_prompts', methods=['POST'])
def generate_prompts():
    data = request.json
    filenames_str = data.get('filenames', '')
    sample_prompt_text = data.get('samplePromptText', '')
    api_key = data.get('apiKey', '')
    api_provider = data.get('apiProvider', '')

    if not all([filenames_str, sample_prompt_text, api_key, api_provider]):
        return jsonify({"error": "Missing required fields"}), 400

    # Temporarily set the API key as an environment variable for the session
    # This is done here for demonstration. In a production setting, keys would be
    # managed more securely (e.g., through a dedicated secrets manager or direct
    # injection into the environment before the app starts).
    if api_provider == "openai":
        os.environ["OPENAI_API_KEY"] = api_key
    elif api_provider == "gemini":
        os.environ["GEMINI_API_KEY"] = api_key
    elif api_provider == "anthropic":
        os.environ["ANTHROPIC_API_KEY"] = api_key
    else:
        return jsonify({"error": "Invalid API provider"}), 400

    filenames = [f.strip() for f in filenames_str.split('\n') if f.strip()]

    if not filenames:
        return jsonify({"error": "No filenames provided"}), 400

    file_handler = FileHandler()
    
    # Ensure a clean slate for each generation
    if os.path.exists(OUTPUT_FOLDER):
        file_handler.clean_output_folder()
    os.makedirs(OUTPUT_FOLDER, exist_ok=True) # Recreate the folder

    generated_count = 0
    errors = []

    for filename in filenames:
        try:
            generator = PromptGenerator(api_provider)
            prompt_content = generator.generate_feature_prompt(filename, sample_prompt_text)
            if prompt_content:
                if file_handler.save_markdown_file(filename, prompt_content):
                    generated_count += 1
                else:
                    errors.append(f"Failed to save {filename}")
            else:
                errors.append(f"Failed to generate content for {filename}")
        except Exception as e:
            errors.append(f"Error processing {filename}: {str(e)}")
        time.sleep(0.5) # Small delay to avoid hitting rate limits too quickly

    if generated_count > 0:
        zip_file_path = file_handler.create_zip_archive()
        if zip_file_path:
            # We will send a success message and then the frontend can trigger the download
            return jsonify({
                "message": f"Successfully generated {generated_count} prompts. Zip file created.",
                "download_ready": True,
                "errors": errors
            }), 200
        else:
            return jsonify({"message": "Generated prompts but failed to create zip.", "errors": errors}), 500
    else:
        return jsonify({"message": "No prompts were generated.", "errors": errors}), 500

@app.route('/download_prompts', methods=['GET'])
def download_prompts():
    zip_path = os.path.join(os.getcwd(), ZIP_FILE_NAME)
    if os.path.exists(zip_path):
        return send_file(zip_path, as_attachment=True, download_name=ZIP_FILE_NAME)
    else:
        return jsonify({"error": "Zip file not found. Generate prompts first."}), 404

if __name__ == '__main__':
    app.run(debug=True, port=5000) # Run on port 5000
```

#### 2\. Frontend Setup (`frontend/`)

**2.1. Create React App**

Navigate to the `feature_prompt_generator` directory in your terminal and run:

```bash
npx create-react-app frontend
cd frontend
npm start
```

This will set up a basic React app and open it in your browser. You can stop it for now.

**2.2. Modify `frontend/src/App.js`**

```javascript
import React, { useState } from 'react';
import './App.css'; // Make sure this is linked if you add styles

function App() {
  const [filenames, setFilenames] = useState('');
  const [samplePromptText, setSamplePromptText] = useState('');
  const [apiKey, setApiKey] = useState('');
  const [apiProvider, setApiProvider] = useState('openai'); // Default to OpenAI
  const [message, setMessage] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [downloadReady, setDownloadReady] = useState(false);
  const [errors, setErrors] = useState([]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setMessage('');
    setDownloadReady(false);
    setErrors([]);
    setIsLoading(true);

    try {
      const response = await fetch('http://127.0.0.1:5000/generate_prompts', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          filenames,
          samplePromptText,
          apiKey,
          apiProvider,
        }),
      });

      const data = await response.json();

      if (response.ok) {
        setMessage(data.message);
        setDownloadReady(data.download_ready);
        setErrors(data.errors || []);
      } else {
        setMessage(`Error: ${data.error || 'Something went wrong.'}`);
        setErrors(data.errors || []);
      }
    } catch (error) {
      console.error('Network error:', error);
      setMessage('Network error. Could not connect to the backend server.');
    } finally {
      setIsLoading(false);
    }
  };

  const handleDownload = () => {
    window.location.href = 'http://127.0.0.1:5000/download_prompts';
  };

  return (
    <div className="App">
      <h1>Feature Prompt Generator</h1>
      <form onSubmit={handleSubmit}>
        <div>
          <label htmlFor="filenames">List of Filenames (one per line, e.g., `feature_login.md`):</label>
          <textarea
            id="filenames"
            rows="10"
            value={filenames}
            onChange={(e) => setFilenames(e.target.value)}
            required
            placeholder="feature_user_registration.md&#x0A;feature_product_catalog.md&#x0A;bugfix_login_issue.md"
          ></textarea>
        </div>
        <div>
          <label htmlFor="samplePromptText">Sample Prompt Text (This will be expanded for each feature):</label>
          <textarea
            id="samplePromptText"
            rows="5"
            value={samplePromptText}
            onChange={(e) => setSamplePromptText(e.target.value)}
            required
            placeholder="Create a Python function to validate user input for a registration form. It should check for strong passwords and valid email formats."
          ></textarea>
        </div>
        <div>
          <label htmlFor="apiKey">API Key:</label>
          <input
            type="password"
            id="apiKey"
            value={apiKey}
            onChange={(e) => setApiKey(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="apiProvider">API Provider:</label>
          <select
            id="apiProvider"
            value={apiProvider}
            onChange={(e) => setApiProvider(e.target.value)}
          >
            <option value="openai">OpenAI</option>
            <option value="gemini">Google Gemini</option>
            <option value="anthropic">Anthropic Claude</option>
          </select>
        </div>
        <button type="submit" disabled={isLoading}>
          {isLoading ? 'Generating...' : 'Generate Prompts'}
        </button>
      </form>

      {message && <p className="message">{message}</p>}
      {errors.length > 0 && (
        <div className="errors">
          <h3>Errors:</h3>
          <ul>
            {errors.map((err, index) => (
              <li key={index}>{err}</li>
            ))}
          </ul>
        </div>
      )}
      {downloadReady && (
        <button onClick={handleDownload} className="download-button">
          Download Prompts (.zip)
        </button>
      )}
    </div>
  );
}

export default App;
```

**2.3. Add some basic styling in `frontend/src/App.css` (optional, but good for readability)**

```css
.App {
  font-family: sans-serif;
  max-width: 800px;
  margin: 40px auto;
  padding: 20px;
  border: 1px solid #ccc;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  background-color: #f9f9f9;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
}

form div {
  margin-bottom: 15px;
}

label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
  color: #555;
}

input[type="text"],
input[type="password"],
textarea,
select {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-sizing: border-box; /* Include padding in width */
  font-size: 16px;
}

textarea {
  resize: vertical;
}

button {
  display: block;
  width: 100%;
  padding: 12px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  font-size: 18px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

button:hover {
  background-color: #0056b3;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.message {
  margin-top: 20px;
  padding: 10px;
  background-color: #e6ffe6;
  border: 1px solid #b3ffb3;
  border-radius: 4px;
  color: #333;
  text-align: center;
}

.errors {
    margin-top: 20px;
    padding: 10px;
    background-color: #ffe6e6;
    border: 1px solid #ffb3b3;
    border-radius: 4px;
    color: #cc0000;
}

.errors ul {
    list-style-type: none;
    padding: 0;
}

.errors li {
    margin-bottom: 5px;
}

.download-button {
  margin-top: 20px;
  background-color: #28a745;
}

.download-button:hover {
  background-color: #218838;
}
```

### How to Run the Application

**1. Install Python Dependencies:**

Navigate to the `feature_prompt_generator` root directory (where `requirements.txt` is) and run:

```bash
pip install -r requirements.txt
```

**2. Start the Backend Server:**

Navigate to the `feature_prompt_generator/backend` directory and run:

```bash
python app.py
```

You should see output similar to `* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)`.

**3. Start the Frontend Development Server:**

Open a *new* terminal, navigate to the `feature_prompt_generator/frontend` directory, and run:

```bash
npm start
```

This will typically open your browser to `http://localhost:3000`.

### Using the Application

1.  **Open your browser** to `http://localhost:3000`.
2.  **Fill in the form fields:**
      * **List of Filenames:** Enter the desired `.md` filenames, one per line (e.g., `feature_auth_service.md`, `feature_data_validation.md`). Remember you wanted around 70, so paste a good number here.
      * **Sample Prompt Text:** Provide a generic prompt that gives the AI context on the *type* of content you expect for a feature-building prompt. The `prompt_generator.py` will then expand upon this for each specific filename.
      * **API Key:** Paste your API key for the chosen provider.
      * **API Provider:** Select "OpenAI", "Google Gemini", or "Anthropic Claude".
3.  **Click "Generate Prompts"**.
4.  The backend will start calling the AI API for each filename. You'll see messages in your backend terminal as files are generated.
5.  Once all prompts are generated (or an error occurs), the frontend will display a message. If successful, a "Download Prompts (.zip)" button will appear.
6.  **Click "Download Prompts (.zip)"** to download the `generated_prompts.zip` file containing all your `.md` prompts.
7.  The `generated_prompts` folder will be created in your `feature_prompt_generator` root directory.

### Automation and "Set It and Forget It"

By running the Flask backend and the React frontend locally, you achieve the "set it and forget it" aspect.

  * You initiate the process from the browser.
  * The Python backend handles the API calls sequentially, saves the files, and zips them.
  * You can then minimize the browser or switch to other tasks while the generation runs in the background on your local machine.
  * Once completed, you'll get a notification in the UI and can download the results.

### Security Considerations

  * **API Keys:**
      * **`.env` file:** Crucial for keeping your API keys out of your source code. The `python-dotenv` library loads these securely.
      * **Never commit `.env`:** Add `.env` to your `.gitignore` file.
      * **Frontend handling:** The API key is sent directly from the frontend to your *local* backend server. This is generally acceptable for a local-only application but would be a severe security risk in a public-facing web application. For local use, it's fine as the key doesn't leave your machine's network.
  * **CORS:** `flask_cors` is used to allow your frontend (running on `localhost:3000`) to communicate with your backend (running on `localhost:5000`). This is essential for local development.
  * **Error Handling:** Basic `try-except` blocks are included for API calls and file operations. You might want to add more granular error logging for debugging.

This setup provides a robust and user-friendly way to automate your feature-building prompt generation. Good luck with your project\!