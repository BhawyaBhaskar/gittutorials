import os
from dotenv import load_dotenv
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient
import fitz  # PyMuPDF for PDFs
import docx  # python-docx for Word
import uuid

# Load environment variables
load_dotenv()
endpoint = os.getenv("AZURE_SEARCH_ENDPOINT")
api_key = os.getenv("AZURE_SEARCH_KEY")
index_name = os.getenv("AZURE_SEARCH_INDEX")

# Validate credentials
if not all([endpoint, api_key, index_name]):
    raise EnvironmentError("Missing required environment variables.")

# Initialize Azure Search Client
search_client = SearchClient(
    endpoint=endpoint,
    index_name=index_name,
    credential=AzureKeyCredential(api_key)
)

# Set path to your folder
folder_path = r"C:\Users\YourName\Documents\MyDocs"  # Update this

# Allowed file types
supported_extensions = [".txt", ".pdf", ".docx"]
documents = []

# Loop through files in the folder
for filename in os.listdir(folder_path):
    file_path = os.path.join(folder_path, filename)
    _, ext = os.path.splitext(filename.lower())

    if ext not in supported_extensions:
        continue

    content = ""
    try:
        if ext == ".txt":
            with open(file_path, "r", encoding="utf-8") as f:
                content = f.read()
        elif ext == ".pdf":
            with fitz.open(file_path) as pdf:
                for page in pdf:
                    content += page.get_text()
        elif ext == ".docx":
            doc = docx.Document(file_path)
            content = "\n".join([para.text for para in doc.paragraphs])
    except Exception as e:
        print(f"Error reading {filename}: {e}")
        continue

    if content:
        documents.append({
            "Id": str(uuid.uuid4()),  # Unique ID
            "title": filename,
            "description": "Auto-uploaded from folder",
            "content": content
        })

# Upload to Azure Search
if documents:
    result = search_client.upload_documents(documents=documents)
    print("\nUpload Results:")
    for r in result:
        status = "Success" if 200 <= r["statusCode"] < 300 else "Failed"
        print(f" - ID: {r['key']} | Status: {status} (Code: {r['statusCode']})")
else:
    print("No valid documents found in the folder.")
