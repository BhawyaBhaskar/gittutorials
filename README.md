from langchain.document_loaders import PyPDFLoader, UnstructuredWordDocumentLoader, TextLoader
from langchain.text_splitter import CharacterTextSplitter
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient

# Azure AI Search service details
service_endpoint = "https://<your-service-name>.search.windows.net"
api_key = "<your-api-key>"
index_name = "<your-index-name>"

# Initialize the SearchClient
credential = AzureKeyCredential(api_key)
search_client = SearchClient(endpoint=service_endpoint, index_name=index_name, credential=credential)

# Define file paths for your documents
pdf_path = "path/to/your/document.pdf"
docx_path = "path/to/your/document.docx"
txt_path = "path/to/your/document.txt"

# Load documents using LangChain loaders
pdf_loader = PyPDFLoader(pdf_path)
docx_loader = UnstructuredWordDocumentLoader(docx_path)
txt_loader = TextLoader(txt_path)

pdf_documents = pdf_loader.load()
docx_documents = docx_loader.load()
txt_documents = txt_loader.load()

# Combine all loaded documents
all_documents = pdf_documents + docx_documents + txt_documents

# Initialize the text splitter
text_splitter = CharacterTextSplitter(
    separator="\n",
    chunk_size=1000,
    chunk_overlap=200
)

# Split documents into chunks
split_documents = text_splitter.split_documents(all_documents)

# Prepare documents for Azure AI Search
search_documents = []
for idx, doc in enumerate(split_documents):
    search_doc = {
        "id": f"doc-{idx}",
        "content": doc.page_content,
        "source": doc.metadata.get("source", "unknown")
    }
    search_documents.append(search_doc)

# Upload documents in batches
batch_size = 1000  # Azure AI Search allows up to 1000 documents per batch
for i in range(0, len(search_documents), batch_size):
    batch = search_documents[i:i + batch_size]
    try:
        result = search_client.upload_documents(documents=batch)
        succeeded = sum(1 for r in result if r.succeeded)
        print(f"Batch {i // batch_size + 1}: {succeeded} documents uploaded successfully.")
    except Exception as e:
        print(f"An error occurred while uploading batch {i // batch_size + 1}: {e}")
