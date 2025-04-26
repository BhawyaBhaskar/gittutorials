import os
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex,
    SimpleField,
    SearchableField,
    ComplexField,
    VectorSearch,
    SearchFieldDataType
)
from azure.core.credentials import AzureKeyCredential
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# Ensure the required environment variables are loaded
search_endpoint = os.getenv("AZURE_SEARCH_ENDPOINT")
search_api_key = os.getenv("AZURE_SEARCH_ADMIN_KEY")
index_name = os.getenv("AZURE_SEARCH_INDEX_NAME")

if not search_endpoint or not search_api_key or not index_name:
    raise ValueError("Missing required environment variables in .env file")

# Initialize the SearchIndexClient to manage indexes
index_client = SearchIndexClient(
    endpoint=search_endpoint,
    credential=AzureKeyCredential(search_api_key)
)

def create_search_index():
    """Create the Azure Cognitive Search index if it does not already exist."""
    # Check if index exists
    try:
        existing_index = index_client.get_index(index_name)
        print(f"Index '{index_name}' already exists.")
        return existing_index
    except Exception:
        print(f"Index '{index_name}' does not exist. Creating a new one.")

    # Define the index schema with appropriate fields
    index = SearchIndex(
        name=index_name,
        fields=[
            SimpleField(name="id", type=SearchFieldDataType.STRING, key=True),
            SearchableField(name="title", type=SearchFieldDataType.STRING),
            SearchableField(name="description", type=SearchFieldDataType.STRING),
            SearchableField(name="content", type=SearchFieldDataType.STRING),
            ComplexField(name="embedding", fields=[
                SimpleField(name="vector", type=SearchFieldDataType.Collection(SearchFieldDataType.Single))
            ])
        ]
    )
    
    # Optionally: Add vector search configuration (for embeddings)
    index.vector_search = VectorSearch(
        algorithm_configurations=[
            {
                "name": "embedding",
                "kind": "hnsw",
                "dimensions": 1536  # Set the dimension based on your embeddings (e.g., OpenAI's embedding model)
            }
        ]
    )

    # Create the index
    try:
        index_client.create_index(index)
        print(f"Index '{index_name}' created successfully.")
    except Exception as e:
        print(f"Error creating index: {e}")
        return None

# Call the function to create the index
if __name__ == "__main__":
    create_search_index()
