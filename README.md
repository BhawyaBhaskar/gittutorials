Function to retrieve document IDs
def get_document_ids(endpoint, api_key, index_name, top=10):
    """
    Retrieves document IDs from an Azure Search index.

    :param endpoint: Azure Search endpoint
    :param api_key: Azure Search admin/query key
    :param index_name: Name of the index
    :param top: Number of documents to retrieve
    :return: List of document IDs
    """
    search_client = SearchClient(
        endpoint=endpoint,
        index_name=index_name,
        credential=AzureKeyCredential(api_key)
    )

    results = search_client.search(search_text="*", top=top, select=["Id"])

    ids = []
    for doc in results:
        ids.append(doc["Id"])
    return ids

# Call the function and print document IDs
doc_ids = get_document_ids(endpoint, api_key, index_name, top=5)
print("Document IDs:")
for doc_id in doc_ids:
    print(f" - {doc_id}")
