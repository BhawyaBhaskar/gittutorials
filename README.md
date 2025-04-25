project_root/
├── vector_engine/
│   ├── __init__.py
│   ├── config.py
│   ├── upload_documents.py
│   ├── vector_search.py
│   └── hybrid_search.py
├── requirements.txt

# --- config.py ---
AZURE_SEARCH_ENDPOINT = "https://<your-search-resource>.search.windows.net"
AZURE_SEARCH_KEY = "<your-search-key>"
AZURE_SEARCH_INDEX = "<your-index-name>"
AZURE_OPENAI_KEY = "<your-shared-openai-api-key>"
AZURE_OPENAI_ENDPOINT = "https://<your-target-uri>.openai.azure.com"
AZURE_OPENAI_DEPLOYMENT = "embedding-model"
AZURE_OPENAI_VERSION = "2023-05-15"

# --- upload_documents.py ---
import os
from langchain_community.document_loaders import (
    PyPDFLoader, TextLoader, Docx2txtLoader, UnstructuredHTMLLoader
)
from langchain.text_splitter import CharacterTextSplitter
from langchain.embeddings import AzureOpenAIEmbeddings
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential
from vector_engine import config

embedding_model = AzureOpenAIEmbeddings(
    openai_api_key=config.AZURE_OPENAI_KEY,
    openai_api_base=config.AZURE_OPENAI_ENDPOINT,
    openai_api_version=config.AZURE_OPENAI_VERSION,
    deployment=config.AZURE_OPENAI_DEPLOYMENT
)

search_client = SearchClient(
    endpoint=config.AZURE_SEARCH_ENDPOINT,
    index_name=config.AZURE_SEARCH_INDEX,
    credential=AzureKeyCredential(config.AZURE_SEARCH_KEY)
)

def load_file(file_path):
    ext = file_path.lower().split('.')[-1]
    if ext == 'pdf':
        return PyPDFLoader(file_path).load()
    elif ext == 'txt':
        return TextLoader(file_path).load()
    elif ext == 'docx':
        return Docx2txtLoader(file_path).load()
    elif ext == 'html':
        return UnstructuredHTMLLoader(file_path).load()
    return []

def chunk_documents(docs):
    splitter = CharacterTextSplitter(chunk_size=500, chunk_overlap=50)
    return splitter.split_documents(docs)

def index_chunks(doc_id, chunks):
    for i, chunk in enumerate(chunks):
        embedding = embedding_model.embed_query(chunk.page_content)
        doc = {
            "id": f"{doc_id}_chunk_{i+1}",
            "title": doc_id,
            "content": chunk.page_content,
            "description": chunk.page_content[:150] + "...",
            "embedding": embedding
        }
        search_client.upload_documents(documents=[doc])

def process_folder(folder_path):
    for file_name in os.listdir(folder_path):
        full_path = os.path.join(folder_path, file_name)
        if os.path.isfile(full_path):
            doc_id = os.path.splitext(file_name)[0]
            docs = load_file(full_path)
            for doc in docs:
                doc.metadata["source"] = doc_id
            chunks = chunk_documents(docs)
            index_chunks(doc_id, chunks)

# --- vector_search.py ---
import requests
import time
from langchain.embeddings import AzureOpenAIEmbeddings
from vector_engine import config

embedding_model = AzureOpenAIEmbeddings(
    openai_api_key=config.AZURE_OPENAI_KEY,
    openai_api_base=config.AZURE_OPENAI_ENDPOINT,
    openai_api_version=config.AZURE_OPENAI_VERSION,
    deployment=config.AZURE_OPENAI_DEPLOYMENT
)

def get_vector_from_query(query):
    return embedding_model.embed_query(query)

def vector_search(query, k=5):
    start_time = time.time()
    embedding = get_vector_from_query(query)

    url = f"{config.AZURE_SEARCH_ENDPOINT}/indexes/{config.AZURE_SEARCH_INDEX}/docs/search?api-version=2023-07-01-Preview"
    headers = {
        "Content-Type": "application/json",
        "api-key": config.AZURE_SEARCH_KEY
    }
    body = {
        "vector": {
            "value": embedding,
            "fields": "embedding",
            "k": k
        },
        "select": "id,title,description,content",
        "top": k
    }

    response = requests.post(url, headers=headers, json=body)
    response.raise_for_status()

    duration = time.time() - start_time
    print(f"[Vector Search] Query took {duration:.2f} seconds")
    return response.json()

# --- hybrid_search.py ---
import requests
import time
from langchain.embeddings import AzureOpenAIEmbeddings
from vector_engine import config

embedding_model = AzureOpenAIEmbeddings(
    openai_api_key=config.AZURE_OPENAI_KEY,
    openai_api_base=config.AZURE_OPENAI_ENDPOINT,
    openai_api_version=config.AZURE_OPENAI_VERSION,
    deployment=config.AZURE_OPENAI_DEPLOYMENT
)

def hybrid_search(query, k=5):
    start_time = time.time()
    embedding = embedding_model.embed_query(query)

    url = f"{config.AZURE_SEARCH_ENDPOINT}/indexes/{config.AZURE_SEARCH_INDEX}/docs/search?api-version=2023-07-01-Preview"
    headers = {
        "Content-Type": "application/json",
        "api-key": config.AZURE_SEARCH_KEY
    }
    body = {
        "search": query,
        "vector": {
            "value": embedding,
            "fields": "embedding",
            "k": k
        },
        "searchFields": "title,content",
        "select": "id,title,description,content",
        "top": k
    }

    response = requests.post(url, headers=headers, json=body)
    response.raise_for_status()

    duration = time.time() - start_time
    print(f"[Hybrid Search] Query took {duration:.2f} seconds")
    return response.json()

# --- requirements.txt ---
langchain
openai
azure-search-documents
unstructured
python-docx
PyMuPDF
