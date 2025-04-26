import os
from dotenv import load_dotenv
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential
from langchain_openai import AzureOpenAIEmbeddings
from utils import read_file
from chunker import advanced_split
from searches.search_functions import handle_search

# Load environment variables from .env file
load_dotenv()

# Initialize Azure Cognitive Search client
search_client = SearchClient(
    endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
    index_name=os.getenv("AZURE_SEARCH_INDEX_NAME"),
    credential=AzureKeyCredential(os.getenv("AZURE_SEARCH_ADMIN_KEY"))
)

# Initialize OpenAI embeddings
embedding_model = AzureOpenAIEmbeddings(
    azure_deployment=os.getenv("AZURE_OPENAI_EMBEDDING_DEPLOYMENT"),
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    api_version=os.getenv("AZURE_OPENAI_API_VERSION")
)

def upload_documents_from_folder(folder_path):
    """Recursively upload documents from any folder on the system."""
    if not os.path.exists(folder_path):
        raise ValueError(f"The specified folder does not exist: {folder_path}")

    all_records = []

    # Walk through all subdirectories and files
    for dirpath, _, filenames in os.walk(folder_path):
        for filename in filenames:
            filepath = os.path.join(dirpath, filename)
            if filepath.endswith((".txt", ".docx", ".pdf")):
                try:
                    content = read_file(filepath)
                    base_id = os.path.splitext(filename)[0]
                    title = base_id
                    description = f"Auto-uploaded file: {filename}"

                    # Split into chunks and generate embeddings for each chunk
                    chunks = advanced_split(content)

                    for idx, chunk in enumerate(chunks):
                        chunk_id = f"{base_id}_chunk_{idx}"
                        vector = embedding_model.embed_query(chunk)

                        record = {
                            "id": chunk_id,
                            "title": title,
                            "description": description,
                            "content": chunk,
                            "embedding": vector
                        }
                        all_records.append(record)
                except Exception as e:
                    print(f"Error reading {filename}: {e}")

    # Batch upload to Azure Cognitive Search
    batch_size = 1000
    total_uploaded = 0
    for i in range(0, len(all_records), batch_size):
        batch = all_records[i:i + batch_size]
        try:
            result = search_client.upload_documents(documents=batch)
            total_uploaded += len(result)
            print(f"Uploaded batch {i // batch_size + 1}: {len(batch)} documents.")
        except Exception as e:
            print(f"Error uploading batch {i // batch_size + 1}: {e}")

    print(f"Total documents uploaded: {total_uploaded} documents.")

    # Optionally, verify the upload by checking document count
    verify_upload(total_uploaded)

def verify_upload(expected_count):
    """Verifies the number of documents in the index."""
    try:
        result = search_client.search("*")  # Query all documents
        actual_count = len(list(result))  # Count the results

        if actual_count == expected_count:
            print(f"Upload successful. {actual_count} documents are now in the index.")
        else:
            print(f"Warning: Only {actual_count} documents found in the index, expected {expected_count}.")
    except Exception as e:
        print(f"Error verifying upload: {e}")

# Automatically trigger document upload when the script is run
if __name__ == "__main__":
    folder_path = "/path/to/your/documents"  # Change this path to your folder's path
    upload_documents_from_folder(folder_path)
