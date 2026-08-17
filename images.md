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

import anthropic
import json
import os


# ============================================================
# STEP 2: Create the client
# ============================================================
# Reads ANTHROPIC_API_KEY from environment automatically.

client = anthropic.Anthropic()

# ============================================================
# STEP 3: Define your tools
# ============================================================
# Each tool has three required fields:
#   name        - what Claude calls it in tool_use blocks
#   description - how Claude decides WHEN to use this tool
#   input_schema - JSON Schema defining the tool's parameters
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
# Claude CANNOT call these functions directly.
#
# Claude REQUESTS a tool call
#       ↓
# Your code executes the tool
#       ↓
# Your code returns the result to Claude
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
        # stop_reason == "end_turn"
        # means Claude is done.
        # Extract the text and return it to the caller.
        # This is the primary loop exit.
        # ====================================================

        if response.stop_reason == "end_turn":
            for block in response.content:
                if block.type == "text":
                    return block.text
            return ""
        # ====================================================
        # TOOL USE
        # ====================================================
        # stop_reason == "tool_use"
        # means Claude wants to call one or more tools.
        # We must:
        # 1. Append Claude's response to history
        # 2. Execute the requested tools
        # 3. Append the tool results to history
        # 4. Call Claude again
        # ====================================================

        if response.stop_reason == "tool_use":
            # ------------------------------------------------
            # APPEND 1:
            # Claude's assistant message This saves Claude's tool request(s) into conversation history.
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
                    print(f"→ Calling tool: " f"{block.name}({block.input})")
                    result = execute_tool(block.name,block.input)
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
            # Tool results These are sent back as a user message.
            # ------------------------------------------------

            messages.append({
                "role": "user",
                "content": tool_results
            })
            # ------------------------------------------------
            # Continue the loop.
            # Claude will now see the tool result and can generate the final answer.
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


# Multi-Agent Systems & Coordinator Patterns

<img width="1790" height="961" alt="image" src="https://github.com/user-attachments/assets/6a252bba-aa8c-486e-8a3a-46b501c64257" />
<img width="1768" height="948" alt="image" src="https://github.com/user-attachments/assets/bf1451ce-a48b-4aad-90f8-e2745fe1e34e" />
<img width="1801" height="982" alt="image" src="https://github.com/user-attachments/assets/d735bef8-c2ef-4100-ba42-80f1fb5a2e5d" />
<img width="1799" height="963" alt="image" src="https://github.com/user-attachments/assets/125e25b6-86df-4eb5-bcf2-2abb79c0950c" />

- If you have to sponge an agent that subagent will be sponged as tool itself.
- Claude SDK particularly defined whenever if are configuring a coordinator agent you add task to its allowed tools.
- if the coordinator has to spawn a subagent it has to do a task tool call with a proper agent definition for each subagent.

<img width="761" height="449" alt="image" src="https://github.com/user-attachments/assets/d937d36b-b31b-4f10-9971-9f1948ed0763" />
<img width="738" height="442" alt="image" src="https://github.com/user-attachments/assets/e1887202-c13b-4bf3-baef-280c598571c2" />
<img width="770" height="488" alt="image" src="https://github.com/user-attachments/assets/8118c27d-a604-427c-9758-c85140873609" />
<img width="1774" height="961" alt="image" src="https://github.com/user-attachments/assets/b1349b55-918a-4969-a0d4-d3637a520f13" />
<img width="1734" height="932" alt="image" src="https://github.com/user-attachments/assets/5f45dcc1-e9ef-4697-8a02-49bdca6e7510" />
<img width="1750" height="968" alt="image" src="https://github.com/user-attachments/assets/58ebb8bc-a188-4b91-9393-8650d0530ca5" />

- subagent starts with blank context, its entire world is what you put into the world.
- It prevents from the context contamination and context loss

<img width="1770" height="981" alt="image" src="https://github.com/user-attachments/assets/2bbefed3-789c-40cb-95b9-6ce39c7d45f6" />
<img width="1233" height="674" alt="image" src="https://github.com/user-attachments/assets/fe429fdb-c7b2-4c00-8c25-9048017e4f8e" />
<img width="1789" height="982" alt="image" src="https://github.com/user-attachments/assets/9d044fcd-5611-41b6-b769-7ecee5e03388" />
<img width="1747" height="953" alt="image" src="https://github.com/user-attachments/assets/f5dee01d-2bfb-4e8a-bec7-1a036af7b9b5" />
<img width="1167" height="632" alt="image" src="https://github.com/user-attachments/assets/b0ddea32-01ff-4358-bcd4-0e92833de599" />

# Subagent Context Passing & Session Management

<img width="1841" height="997" alt="image" src="https://github.com/user-attachments/assets/e0cb490e-51c6-4787-a588-fead2267d1d8" />
<img width="1834" height="991" alt="image" src="https://github.com/user-attachments/assets/0c3f675e-78c0-4b93-91c8-ebafe9c28759" />
<img width="1773" height="970" alt="image" src="https://github.com/user-attachments/assets/4f3a57f0-2f4b-499f-a489-dcf648499f20" />

<img width="1210" height="441" alt="image" src="https://github.com/user-attachments/assets/15a5a986-798b-4437-8d27-bc6e9287a71f" />
<img width="557" height="408" alt="image" src="https://github.com/user-attachments/assets/a678ad86-eaa6-485d-89cb-7b0d3b4607a2" />
<img width="1813" height="991" alt="image" src="https://github.com/user-attachments/assets/4c952431-7664-4717-8e1a-3cdc2eace276" />

<img width="1129" height="425" alt="image" src="https://github.com/user-attachments/assets/3ff010f9-eb07-46b8-a137-900cefca4a51" />
<img width="1073" height="394" alt="image" src="https://github.com/user-attachments/assets/87229b88-fee5-4a85-807d-95f817caf89b" />
<img width="465" height="86" alt="image" src="https://github.com/user-attachments/assets/153eda02-a7d6-4529-aff5-e66482f68e2e" />

<img width="1807" height="1001" alt="image" src="https://github.com/user-attachments/assets/af06eac2-26f9-462e-9012-1b3b72a48e46" />
<img width="873" height="369" alt="image" src="https://github.com/user-attachments/assets/f2c06f6f-7469-476e-bbe7-573c4b9194da" />

<img width="1842" height="974" alt="image" src="https://github.com/user-attachments/assets/2a87894c-34c7-4699-82ef-bdf09617a38d" />

<img width="863" height="319" alt="image" src="https://github.com/user-attachments/assets/c0ae9044-28ae-44bc-81fb-1263d318a78f" />
<img width="910" height="479" alt="image" src="https://github.com/user-attachments/assets/ea2beea7-ed2f-4d45-834a-41620f4f970f" />

- you resume when prior context is valid
- Start fresh when your results are stale, you can give a summary what happened

<img width="791" height="354" alt="image" src="https://github.com/user-attachments/assets/099e2598-73e6-4080-b159-554d50533f9c" />

<img width="1779" height="973" alt="image" src="https://github.com/user-attachments/assets/27569b60-4a3a-45f0-a676-1c3fa5d44a32" />
<img width="1815" height="984" alt="image" src="https://github.com/user-attachments/assets/5b8bb318-4147-4dff-b155-d0dd29f1fc73" />
<img width="1852" height="1017" alt="image" src="https://github.com/user-attachments/assets/29cb34d7-50db-4404-b53f-5d0eab5aac80" />

<img width="1750" height="913" alt="image" src="https://github.com/user-attachments/assets/71c9ee55-a475-41ed-8d25-f2daced8e50d" />
<img width="970" height="669" alt="image" src="https://github.com/user-attachments/assets/b3c8d3d7-90ff-4e3d-a280-0b1d043394e1" />

<img width="1236" height="658" alt="image" src="https://github.com/user-attachments/assets/5c550517-f310-4d4f-9595-d55e6e43c21b" />
<img width="1229" height="649" alt="image" src="https://github.com/user-attachments/assets/a0369630-d59a-44a6-b0e3-24aa0dc2a871" />
<img width="1191" height="645" alt="image" src="https://github.com/user-attachments/assets/0dba4521-e87b-4b85-af05-c7ef157b8944" />








