import os
from dotenv import load_dotenv
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient

# Load environment variables
load_dotenv()
endpoint = os.getenv("AZURE_SEARCH_ENDPOINT")
api_key = os.getenv("AZURE_SEARCH_KEY")
index_name = os.getenv("AZURE_SEARCH_INDEX")

# Initialize SearchClient
search_client = SearchClient(
    endpoint=endpoint,
    index_name=index_name,
    credential=AzureKeyCredential(api_key)
)

# Step 1: Fetch all document IDs
print("Fetching document IDs...")
all_ids = []
results = search_client.search(search_text="*", select=["Id"], top=1000)

for doc in results:
    all_ids.append({"Id": doc["Id"]})  # 'Id' must match your index's key field name

print(f"Found {len(all_ids)} documents to delete.")

# Step 2: Delete all documents by ID
if all_ids:
    result = search_client.delete_documents(documents=all_ids)

    print("\nDelete Results:")
    for r in result:
        status = "Success" if 200 <= r["statusCode"] < 300 else "Failed"
        print(f" - ID: {r['key']} | Status: {status} (Code: {r['statusCode']})")
else:
    print("No documents found to delete.")
