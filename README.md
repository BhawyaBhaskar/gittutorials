# vector_search.py

import time
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient

# ========== CONFIG ==========

AZURE_SEARCH_ENDPOINT = "https://<your-search-service>.search.windows.net"
AZURE_SEARCH_KEY = "<your-admin-key>"
AZURE_SEARCH_INDEX_NAME = "your-index-name"
AZURE_EMBEDDING_DEPLOYMENT = "<your-embedding-deployment-name>"

credential = AzureKeyCredential(AZURE_SEARCH_KEY)
search_client = SearchClient(endpoint=AZURE_SEARCH_ENDPOINT, index_name=AZURE_SEARCH_INDEX_NAME, credential=credential)

# ========== VECTOR SEARCH FUNCTION ==========

def vector_search(user_query, k=5):
    # Generate the embedding for the user query
    embedding_response = search_client.embed_query(
        search_text=user_query,
        deployment_name=AZURE_EMBEDDING_DEPLOYMENT,
        embedding_field_name="contentVector"
    )
    vector = embedding_response["embedding"]

    # Perform vector search
    start_time = time.time()
    
    results = search_client.search(
        search_text=None,  # Vector-only search => no keyword text
        vector={
            "value": vector,
            "k": k,
            "fields": "contentVector"
        },
        top=k
    )

    end_time = time.time()

    print(f"\nTop {k} Vector Search Results:")
    for idx, result in enumerate(results):
        print(f"{idx + 1}. ID: {result['id']}")
        print(f"   Content: {result['content'][:200]}...\n")  # Show first 200 characters

    print(f"Time taken for vector search: {end_time - start_time:.2f} seconds")

# ========== MAIN ==========

if __name__ == "__main__":
    query = input("Enter your query for vector search: ")
    vector_search(query)
