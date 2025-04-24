import os
from langchain_community.document_loaders import (
    PyPDFLoader, TextLoader, Docx2txtLoader, UnstructuredHTMLLoader
)
from langchain.text_splitter import CharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential
from config import AZURE_SEARCH_ENDPOINT, AZURE_SEARCH_KEY, AZURE_SEARCH_INDEX, OPENAI_API_KEY

# Clients
embedding_model = OpenAIEmbeddings(openai_api_key=OPENAI_API_KEY)
search_client = SearchClient(
    endpoint=AZURE_SEARCH_ENDPOINT,
    index_name=AZURE_SEARCH_INDEX,
    credential=AzureKeyCredential(AZURE_SEARCH_KEY)
)

# Loaders
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
    else:
        return []

# Split text into chunks
def chunk_documents(docs):
    splitter = CharacterTextSplitter(
        separator="\n", chunk_size=500, chunk_overlap=50
    )
    return splitter.split_documents(docs)

# Upload chunks to Azure Search
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

# Main process
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

if __name__ == "__main__":
    process_folder("/path/to/your/folder")
