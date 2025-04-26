import re

def advanced_split(text, max_length=1000):
    """Splits text into chunks of up to max_length, ensuring no word is cut off."""
    words = text.split()
    chunks = []
    chunk = ""

    for word in words:
        if len(chunk) + len(word) + 1 > max_length:
            chunks.append(chunk)
            chunk = word
        else:
            if chunk:
                chunk += " "
            chunk += word

    if chunk:
        chunks.append(chunk)
    
    return chunks
