# create_index.py

import time
from azure.core.credentials import AzureKeyCredential
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex,
    SearchField,
    SearchFieldDataType,
    SimpleField,
    SearchableField,
    SearchFieldVectorSearchConfiguration,
    VectorSearch,
    VectorSearchProfile,
    ExhaustiveKnnAlgorithmConfiguration,
)

# ========== CONFIG ==========

AZURE_SEARCH_ENDPOINT = "https://<your-search-service>.search.windows.net"
AZURE_SEARCH_KEY = "<your-admin-key>"
AZURE_SEARCH_INDEX_NAME = "your-index-name"

credential = AzureKeyCredential(AZURE_SEARCH_KEY)
index_client = SearchIndexClient(endpoint=AZURE_SEARCH_ENDPOINT, credential=credential)

# ========== FUNCTION TO CREATE INDEX ==========

def create_index():
    start_time = time.time()

    fields = [
        SimpleField(name="id", type=SearchFieldDataType.String, key=True),
        SearchableField(name="content", type=SearchFieldDataType.String),
        SearchField(
            name="contentVector",
            type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
            vector_search_dimensions=1536,
            vector_search_configuration="exhaustive-knn-config"
        )
    ]

    vector_search = VectorSearch(
        profiles=[
            VectorSearchProfile(
                name="exhaustive-knn-profile",
                algorithm_configuration_name="exhaustive-knn-config"
            )
        ],
        algorithms=[
            ExhaustiveKnnAlgorithmConfiguration(
                name="exhaustive-knn-config"
            )
        ]
    )

    index = SearchIndex(
        name=AZURE_SEARCH_INDEX_NAME,
        fields=fields,
        vector_search=vector_search
    )

    # Delete if exists
    try:
        index_client.delete_index(AZURE_SEARCH_INDEX_NAME)
        print(f"Deleted existing index: {AZURE_SEARCH_INDEX_NAME}")
    except Exception:
        print(f"No existing index found: {AZURE_SEARCH_INDEX_NAME}")

    index_client.create_index(index)
    print(f"Created new index: {AZURE_SEARCH_INDEX_NAME}")

    end_time = time.time()
    print(f"Time taken to create index: {end_time - start_time:.2f} seconds")

if __name__ == "__main__":
    create_index()
