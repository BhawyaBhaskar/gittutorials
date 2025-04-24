import os
from langchain.document_loaders import PyPDFLoader, TextLoader, UnstructuredWordDocumentLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential

# Azure Search credentials
AZURE_SEARCH_ENDPOINT = "https://<your-search-service>.search.windows.net"
AZURE_SEARCH_KEY = "<your-key>"
AZURE_SEARCH_INDEX = "<your-index-name>"

# Initialize SearchClient
search_client = SearchClient(
    endpoint=AZURE_SEARCH_ENDPOINT,
    index_name=AZURE_SEARCH_INDEX,
    credential=AzureKeyCredential(AZURE_SEARCH_KEY)
)

# Directory containing your documents
folder_path = "/path/to/your/folder"

# File loader mapping
def load_file(file_path):
    ext = file_path.lower().split('.')[-1]
    if ext == 'pdf':
        return PyPDFLoader(file_path).load()
    elif ext == 'txt':
        return TextLoader(file_path).load()
    elif ext in ['doc', 'docx']:
        return UnstructuredWordDocumentLoader(file_path).load()
    else:
        return []

# Chunking method
def chunk_documents(docs):
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=50
    )
    return splitter.split_documents(docs)

# Index each chunk
def index_chunks(chunks):
    for i, chunk in enumerate(chunks):
        doc = {
            "id": f"doc_{i}",
            "content": chunk.page_content,
            "metadata": chunk.metadata
        }
        search_client.upload_documents(documents=[doc])

# Main process
for file_name in os.listdir(folder_path):
    full_path = os.path.join(folder_path, file_name)
    if os.path.isfile(full_path):
        docs = load_file(full_path)
        chunks = chunk_documents(docs)
        index_chunks(chunks)
