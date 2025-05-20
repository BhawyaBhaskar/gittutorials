from openai import AzureOpenAI
import asyncio
import json
from your_tools_file import tools, tool_functions  # Assume tools and tool handlers are imported

client = AzureOpenAI(
    api_key="YOUR_KEY",
    azure_endpoint="https://YOUR_ENDPOINT.openai.azure.com/",
    api_version="2023-05-15"
)

deployment_name = "your-deployment-name"

print("🤖 Agent ready. Type 'exit' to stop.")

while True:
    user_input = input("You: ")
    if user_input.lower() in ["exit", "quit"]:
        break

    messages.append({"role": "user", "content": user_input})

    # First call — see if tool call is needed
    response = client.chat.completions.create(
        model=deployment_name,
        messages=messages,
        tools=tools,
        tool_choice="auto"
    )
    response_message = response.choices[0].message
    messages.append(response_message)

    # If the model decided to call a function/tool
    if hasattr(response_message, "tool_calls"):
        for tool_call in response_message.tool_calls:
            function_name = tool_call.function.name
            function_args = json.loads(tool_call.function.arguments)

            tool_func = tool_functions.get(function_name)
            if not tool_func:
                tool_response = json.dumps({"error": f"Unknown function {function_name}"})
            else:
                tool_response = await tool_func(**function_args)

            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "name": function_name,
                "content": tool_response,
            })

        # Final call — respond naturally
        final_response = client.chat.completions.create(
            model=deployment_name,
            messages=messages
        )
        final_message = final_response.choices[0].message
        messages.append(final_message)
        print("Agent:", final_message.content)

    else:
        # No tool was called, return direct answer
        print("Agent:", response_message.content)
