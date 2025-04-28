import os
import time
import openai
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential
from docx import Document
import fitz  # PyMuPDF for PDFs
from bs4 import BeautifulSoup

# ===== CONFIG =====

openai.api_type = "azure"
openai.api_key = "YOUR_OPENAI_KEY"
openai.api_version = "2023-05-15"
openai.azure_endpoint = "https://YOUR-RESOURCE-NAME.openai.azure.com/"
DEPLOYMENT_NAME = "YOUR_DEPLOYMENT_NAME"

AZURE_SEARCH_ENDPOINT = "https://YOUR-SEARCH-NAME.search.windows.net"
AZURE_SEARCH_KEY = "YOUR-SEARCH-ADMIN-KEY"
AZURE_SEARCH_INDEX = "YOUR-INDEX-NAME"

# ===== CLIENTS =====

search_client = SearchClient(
    endpoint=AZURE_SEARCH_ENDPOINT,
    index_name=AZURE_SEARCH_INDEX,
    credential=AzureKeyCredential(AZURE_SEARCH_KEY)
)

# ===== HELPERS =====

def get_embedding(text: str) -> list:
    response = openai.Embedding.create(
        input=text,
        model=DEPLOYMENT_NAME
    )
    return response['data'][0]['embedding']

def chunk_text(text, chunk_size=500, overlap=50):
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start += chunk_size - overlap
    return chunks

def load_txt(path):
    with open(path, 'r', encoding='utf-8') as f:
        return f.read()

def load_docx(path):
    doc = Document(path)
    return "\n".join([p.text for p in doc.paragraphs])

def load_pdf(path):
    doc = fitz.open(path)
    return "\n".join([page.get_text() for page in doc])

def load_html(path):
    with open(path, 'r', encoding='utf-8') as f:
        soup = BeautifulSoup(f, 'html.parser')
        return soup.get_text()

def load_document(path):
    ext = path.split('.')[-1].lower()
    if ext == 'txt':
        return load_txt(path)
    elif ext == 'docx':
        return load_docx(path)
    elif ext == 'pdf':
        return load_pdf(path)
    elif ext == 'html':
        return load_html(path)
    else:
        return ""

def index_document_chunks(file_path):
    doc_id = os.path.splitext(os.path.basename(file_path))[0]
    raw_text = load_document(file_path)
    chunks = chunk_text(raw_text)

    documents = []
    for i, chunk in enumerate(chunks):
        if not chunk.strip():
            continue
        embedding = get_embedding(chunk)
        doc = {
            "id": f"{doc_id}_chunk_{i+1}",
            "title": doc_id,
            "content": chunk,
            "description": chunk[:150],
            "embedding": embedding
        }
        documents.append(doc)

    result = search_client.upload_documents(documents=documents)
    print(f"Uploaded {len(documents)} chunks from {file_path}. Result: {result}")

def process_folder(folder_path):
    for file in os.listdir(folder_path):
        full_path = os.path.join(folder_path, file)
        if os.path.isfile(full_path):
            index_document_chunks(full_path)

# ===== SEARCHES =====

def vector_search(query: str):
    vector = get_embedding(query)

    start_time = time.time()

    results = search_client.search(
        search_text=None,
        vector={
            "value": vector,
            "k": 5,
            "fields": "embedding"
        },
        top=5
    )

    end_time = time.time()

    print(f"Vector search took {end_time - start_time:.3f} seconds")

    for result in results:
        print(f"ID: {result['id']}")
        print(f"Content: {result['content'][:200]}")
        print("---")

def hybrid_search(query: str):
    vector = get_embedding(query)

    start_time = time.time()

    results = search_client.search(
        search_text=query,
        vector={
            "value": vector,
            "k": 5,
            "fields": "embedding"
        },
        top=5,
        query_type="semantic"  # optional, only if you have semantic config
    )

    end_time = time.time()

    print(f"Hybrid search took {end_time - start_time:.3f} seconds")

    for result in results:
        print(f"ID: {result['id']}")
        print(f"Content: {result['content'][:200]}")
        print("---")

# ====== USAGE ======

# Example usage
# process_folder("./docs")  # <-- Your folder with documents
# vector_search("What is AI?")
# hybrid_search("Benefits of machine learning")
