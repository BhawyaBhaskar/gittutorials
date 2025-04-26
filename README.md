from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential
import time
from dotenv import load_dotenv
import os

# Load environment variables from .env file
load_dotenv()

# Initialize the Azure Cognitive Search client
search_client = SearchClient(
    endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
    index_name=os.getenv("AZURE_SEARCH_INDEX_NAME"),
    credential=AzureKeyCredential(os.getenv("AZURE_SEARCH_ADMIN_KEY"))
)

def handle_search(query):
    """Handle full-text, vector, and hybrid searches."""
    print("Performing Full-Text Search:")
    start_time = time.time()
    full_text_results = perform_full_text_search(query)
    print(f"Full-Text Search Results: {full_text_results}")
    print(f"Full-Text Search took {time.time() - start_time:.2f} seconds.")

    print("\nPerforming Vector Search:")
    start_time = time.time()
    vector_results = perform_vector_search(query)
    print(f"Vector Search Results: {vector_results}")
    print(f"Vector Search took {time.time() - start_time:.2f} seconds.")

    print("\nPerforming Hybrid Search:")
    start_time = time.time()
    hybrid_results = perform_hybrid_search(query)
    print(f"Hybrid Search Results: {hybrid_results}")
    print(f"Hybrid Search took {time.time() - start_time:.2f} seconds.")

def perform_full_text_search(query):
    """Perform a full-text search."""
    results = search_client.search(query)
    return [result['title'] for result in results]

def perform_vector_search(query):
    """Perform a vector search."""
    vector = query  # For simplicity, assume query is a vector. Replace with actual vector logic.
    results = search_client.search(query)  # Replace with actual vector search logic.
    return [result['title'] for result in results]

def perform_hybrid_search(query):
    """Perform a hybrid search combining full-text and vector search."""
    full_text_results = perform_full_text_search(query)
    vector_results = perform_vector_search(query)
    return list(set(full_text_results + vector_results))
