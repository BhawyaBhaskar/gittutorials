async def process_log_data(log_data: str) -> str:
    return f"Processed log data (preview): {log_data[:100]}..."

async def analyze_dependency_data(log: str, table: str) -> str:
    return f"Analyzed dependency table '{table}' from log."

tool_functions = {
    "process_log_data": process_log_data,
    "analyze_dependency_data": analyze_dependency_data,
}

tool_schemas = [
    {
        "type": "function",
        "function": {
            "name": "process_log_data",
            "description": "Processes raw log data",
            "parameters": {
                "type": "object",
                "properties": {
                    "log_data": {"type": "string"},
                },
                "required": ["log_data"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "analyze_dependency_data",
            "description": "Analyzes dependency data from logs",
            "parameters": {
                "type": "object",
                "properties": {
                    "log": {"type": "string"},
                    "table": {"type": "string"},
                },
                "required": ["log", "table"]
            }
        }
    }
]
