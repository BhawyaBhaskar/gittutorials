from azure.search.documents import SearchIndexClient
from azure.search.documents.models import *
from azure.core.credentials import AzureKeyCredential
import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Set up credentials from .env
search_service_name = os.getenv('SEARCH_SERVICE_NAME')
search_api_key = os.getenv('SEARCH_API_KEY')
index_name = os.getenv('SEARCH_INDEX_NAME')

# Initialize the SearchIndexClient
endpoint = f"https://{search_service_name}.search.windows.net"
credential = AzureKeyCredential(search_api_key)
client = SearchIndexClient(endpoint=endpoint, credential=credential)

def create_index():
    # Define the index schema
    index = SearchIndex(name=index_name)
    index.fields = [
        SearchField(name="id", type=SearchFieldDataType.String, is_key=True, is_filterable=True),
        SearchField(name="title", type=SearchFieldDataType.String, is_searchable=True, is_filterable=True),
        SearchField(name="content", type=SearchFieldDataType.String, is_searchable=True),
        SearchField(name="description", type=SearchFieldDataType.String, is_searchable=True),
        SearchField(name="embedding", type=SearchFieldDataType.Collection(SearchFieldDataType.Single), is_searchable=True, is_filterable=False)
    ]
    
    # Create the index
    client.create_index(index)
    print(f"Index '{index_name}' created successfully!")

if __name__ == "__main__":
    create_index()
