<img width="1906" height="947" alt="image" src="https://github.com/user-attachments/assets/9dbf851f-919b-47b5-916f-8695cf06d6ce" />
<img width="715" height="369" alt="image" src="https://github.com/user-attachments/assets/13154e01-35ed-4bc3-ae1b-d47178abc47c" />

## The roles in the Claude 
- System role
- User role
- Assistant role

> each message should be associated with role.
> in GUI we do not need to mention the role. by default, its user
> But in Claude SDK we need to specify.

<img width="1786" height="952" alt="image" src="https://github.com/user-attachments/assets/2a6242e5-7b21-49e5-b9e2-ad1e86cfdd91" />

- Agents are always associated with some tools. 

<img width="1822" height="975" alt="image" src="https://github.com/user-attachments/assets/a983393a-790f-4da6-b4db-fda3002a6b1b" />
<img width="1813" height="962" alt="image" src="https://github.com/user-attachments/assets/919c490d-b1f1-4378-a505-8cae5ac92e8e" />

```python
# ============================================================
# STEP 1: Install and import
# ============================================================
# pip install anthropic

import anthropic
import json
import os


# ============================================================
# STEP 2: Create the client
# ============================================================
# Reads ANTHROPIC_API_KEY from environment automatically.
# Never hardcode the key here.

client = anthropic.Anthropic()


# ============================================================
# STEP 3: Define your tools
# ============================================================
#
# Each tool has three required fields:
#
#   name        - what Claude calls it in tool_use blocks
#   description - how Claude decides WHEN to use this tool
#   input_schema - JSON Schema defining the tool's parameters
#
# ============================================================

tools = [
    {
        "name": "lookup_order",

        "description": (
            "Look up an order by its order ID. "
            "Returns current status, estimated delivery date, and carrier name. "
            "Use this when the customer asks where their order is "
            "or when it will arrive."
        ),

        "input_schema": {
            "type": "object",

            "properties": {
                "order_id": {
                    "type": "string",
                    "description": "The numeric order ID (e.g. '4821')"
                }
            },

            "required": ["order_id"]
        }
    }
]


# ============================================================
# STEP 4: Implement the actual tools
# ============================================================
#
# Claude CANNOT call these functions directly.
#
# Claude REQUESTS a tool call
#       ↓
# Your code executes the tool
#       ↓
# Your code returns the result to Claude
#
# This is a mock implementation.
# In production, this would query a real database/API.
#
# ============================================================

def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Run a tool and return its result as a JSON string."""

    if tool_name == "lookup_order":

        order_id = tool_input.get("order_id", "")

        # Mock database lookup
        mock_orders = {
            "4821": {
                "status": "shipped",
                "eta": "March 30",
                "carrier": "FedEx"
            },

            "9910": {
                "status": "processing",
                "eta": "April 2",
                "carrier": "UPS"
            },

            "0042": {
                "status": "delivered",
                "eta": "March 25",
                "carrier": "DHL"
            }
        }

        if order_id in mock_orders:
            return json.dumps(mock_orders[order_id])

        else:
            return json.dumps({
                "error": f"Order {order_id} not found"
            })

    # Unknown tool - return an error result
    # Never raise an exception here
    return json.dumps({
        "error": f"Unknown tool: {tool_name}"
    })


# ============================================================
# STEP 5: Run the Claude agent
# ============================================================

def run_agent(user_message: str) -> str:

    # Conversation history
    messages = [
        {
            "role": "user",
            "content": user_message
        }
    ]

    # --------------------------------------------------------
    # Agent loop
    # --------------------------------------------------------
    while True:

        response = client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        # ====================================================
        # EXIT CONDITION
        # ====================================================
        #
        # stop_reason == "end_turn"
        # means Claude is done.
        #
        # Extract the text and return it to the caller.
        # This is the primary loop exit.
        #
        # ====================================================

        if response.stop_reason == "end_turn":

            for block in response.content:

                if block.type == "text":
                    return block.text

            # end_turn with no text
            # rare but possible
            return ""

        # ====================================================
        # TOOL USE
        # ====================================================
        #
        # stop_reason == "tool_use"
        # means Claude wants to call one or more tools.
        #
        # We must:
        #
        # 1. Append Claude's response to history
        # 2. Execute the requested tools
        # 3. Append the tool results to history
        # 4. Call Claude again
        #
        # ====================================================

        if response.stop_reason == "tool_use":

            # ------------------------------------------------
            # APPEND 1:
            # Claude's assistant message
            #
            # This saves Claude's tool request(s)
            # into conversation history.
            #
            # MUST happen before tool results.
            # ------------------------------------------------

            messages.append({
                "role": "assistant",
                "content": response.content
            })

            # ------------------------------------------------
            # Execute each tool Claude requested
            # ------------------------------------------------

            tool_results = []

            for block in response.content:

                if block.type == "tool_use":

                    print(
                        f"→ Calling tool: "
                        f"{block.name}({block.input})"
                    )

                    result = execute_tool(
                        block.name,
                        block.input
                    )

                    print(f"→ Result: {result}")

                    # ----------------------------------------
                    # Create tool_result block
                    # ----------------------------------------

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # ------------------------------------------------
            # APPEND 2:
            # Tool results
            #
            # These are sent back as a user message.
            # ------------------------------------------------

            messages.append({
                "role": "user",
                "content": tool_results
            })

            # ------------------------------------------------
            # Continue the loop.
            #
            # Claude will now see the tool result and can
            # generate the final answer.
            # ------------------------------------------------

            continue


# ============================================================
# STEP 6: Test the agent
# ============================================================

if __name__ == "__main__":

    user_message = input("You: ")

    answer = run_agent(user_message)

    print("\nClaude:")
    print(answer)
```

<img width="1805" height="966" alt="image" src="https://github.com/user-attachments/assets/1b3f4eda-5f5d-4fa0-b953-28c20484fe5b" />
<img width="1804" height="958" alt="image" src="https://github.com/user-attachments/assets/38a77381-48f6-48e9-8edc-6e710bce3064" />


<img width="1833" height="938" alt="image" src="https://github.com/user-attachments/assets/1da801da-42f5-4459-b2f1-6e614f68e5d6" />



<img width="1912" height="1019" alt="image" src="https://github.com/user-attachments/assets/659bdbec-3483-4482-b4fa-a8a388ef3df8" />

<img width="1729" height="923" alt="image" src="https://github.com/user-attachments/assets/21fdcc31-e7aa-4392-b94f-64c6ad9cf73f" />

<img width="1798" height="948" alt="image" src="https://github.com/user-attachments/assets/76b33096-a560-46bc-b279-40ecd580c923" />

<img width="1796" height="965" alt="image" src="https://github.com/user-attachments/assets/9c3073f1-714f-49ff-9cbe-79f0703dbff9" />

<img width="1806" height="930" alt="image" src="https://github.com/user-attachments/assets/b707eda7-0aaf-48d2-8159-df2203e95f58" />

<img width="1785" height="980" alt="image" src="https://github.com/user-attachments/assets/2d0b7c32-7dba-4625-9f5f-a865b07bbd8a" />







