import os
import time
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex,
    SearchField,
    SearchFieldDataType,
    SimpleField,
    SearchableField,
    VectorSearch,
    HnswAlgorithmConfiguration,
    VectorSearchProfile,
)
from docx import Document
import fitz  # PyMuPDF
from typing import List
from config import AZURE_SEARCH_ENDPOINT, AZURE_SEARCH_KEY, AZURE_SEARCH_INDEX_NAME

# Initialize clients
search_credential = AzureKeyCredential(AZURE_SEARCH_KEY)
search_client = SearchClient(endpoint=AZURE_SEARCH_ENDPOINT, index_name=AZURE_SEARCH_INDEX_NAME, credential=search_credential)
index_client = SearchIndexClient(endpoint=AZURE_SEARCH_ENDPOINT, credential=search_credential)

# Function to load DOCX
def load_docx(file_path: str) -> str:
    doc = Document(file_path)
    return "\n".join([para.text for para in doc.paragraphs])

# Function to load PDF
def load_pdf(file_path: str) -> str:
    doc = fitz.open(file_path)
    return "\n".join([page.get_text() for page in doc])

# Function to load TXT
def load_txt(file_path: str) -> str:
    with open(file_path, 'r', encoding='utf-8') as f:
        return f.read()

# General loader based on file extension
def load_document(file_path: str) -> str:
    ext = os.path.splitext(file_path)[1].lower()
    if ext == ".pdf":
        return load_pdf(file_path)
    elif ext == ".docx":
        return load_docx(file_path)
    elif ext == ".txt":
        return load_txt(file_path)
    else:
        print(f"Unsupported file type: {ext}")
        return ""

# Split text into chunks
def split_text(text: str, chunk_size=500, overlap=50) -> List[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = min(start + chunk_size, len(text))
        chunks.append(text[start:end])
        start += chunk_size - overlap
    return chunks

# Function to create the index
def create_index():
    fields = [
        SimpleField(name="id", type=SearchFieldDataType.String, key=True),
        SearchableField(name="title", type=SearchFieldDataType.String),
        SearchableField(name="content", type=SearchFieldDataType.String),
        SearchField(name="contentVector", type=SearchFieldDataType.Collection(SearchFieldDataType.Single), vector_search_dimensions=1536, vector_search_profile_name="default"),
    ]

    vector_search = VectorSearch(
        profiles=[VectorSearchProfile(name="default", algorithm_configuration_name="default")],
        algorithms=[HnswAlgorithmConfiguration(name="default")]
    )

    index = SearchIndex(
        name=AZURE_SEARCH_INDEX_NAME,
        fields=fields,
        vector_search=vector_search
    )

    # Delete if exists
    if index_client.get_index(AZURE_SEARCH_INDEX_NAME):
        index_client.delete_index(AZURE_SEARCH_INDEX_NAME)

    index_client.create_index(index)
    print(f"Index '{AZURE_SEARCH_INDEX_NAME}' created successfully.")

# Function to upload documents
def upload_documents(folder_path: str):
    for filename in os.listdir(folder_path):
        filepath = os.path.join(folder_path, filename)
        if not os.path.isfile(filepath):
            continue

        text = load_document(filepath)
        if not text:
            continue

        chunks = split_text(text)

        documents = []
        doc_id_base = os.path.splitext(filename)[0]

        for i, chunk in enumerate(chunks):
            documents.append({
                "id": f"{doc_id_base}_chunk_{i}",
                "title": doc_id_base,
                "content": chunk,
                "contentVector": None  # Let Azure AI Search auto-generate embeddings
            })

        result = search_client.upload_documents(documents)
        print(f"Uploaded {len(documents)} documents from {filename}. Result: {result}")

# Hybrid Search (Text + Vector)
def hybrid_search(query: str):
    from datetime import datetime
    start_time = datetime.now()

    results = search_client.search(
        search_text=query,
        vector={"value": None, "fields": "contentVector", "kNearestNeighborsCount": 3},
        top=3
    )

    for doc in results:
        print(doc)

    end_time = datetime.now()
    print(f"Query took: {(end_time - start_time).total_seconds()} seconds.")

# Vector-only Search
def vector_search(vector: List[float]):
    from datetime import datetime
    start_time = datetime.now()

    results = search_client.search(
        search_text=None,
        vector={"value": vector, "fields": "contentVector", "kNearestNeighborsCount": 3},
        top=3
    )

    for doc in results:
        print(doc)

    end_time = datetime.now()
    print(f"Vector Query took: {(end_time - start_time).total_seconds()} seconds.")

