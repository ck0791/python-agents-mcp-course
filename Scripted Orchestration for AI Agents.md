
## Intelligence Budget: Scripted Orchestration

The Multi-Step Problem

Real workflows often span multiple operations. You need to get data from one source, transform it, combine it with data from another source, apply business logic, and produce a result. The obvious approach is to have the agent orchestrate each step—call a tool, look at the result, decide what to do next, call another tool, and so on.

This works, but it’s expensive. Every intermediate result flows through the agent’s context. Every step requires the agent to reason about what it sees. For data-intensive workflows, you end up paying for the agent to process information that it doesn’t really need to think about—it’s just routing data from one place to another.

Consider this scenario: a user asks, “Generate a report of all customers who haven’t ordered in 90 days and check if they have open support tickets.”

The step-by-step approach looks like this:

Agent: "I'll get the customer list first."
Agent calls: get_all_customers()
Tool returns: [... 2,847 customer records ...]

Agent: "Now I need to check order history for each..."
Agent calls: get_orders(customer_id: "cust_001")
Tool returns: [... order history ...]

Agent: "That customer ordered recently. Next customer..."
Agent calls: get_orders(customer_id: "cust_002")
Tool returns: [... order history ...]

... (2,845 more iterations) ...

The agent sees every customer record. It sees every order history. It reasons about each one individually. For 2,847 customers, that’s thousands of tool calls and potentially hundreds of thousands of tokens flowing through the context.

The problem isn’t that this doesn’t work—it does. The problem is that the agent is doing mechanical work. It’s not making interesting decisions; it’s just iterating through a list and checking conditions. That’s exactly the kind of work that code does well and cheaply.
Scripted Orchestration

Here’s a different approach: instead of calling each tool and examining the result, the agent writes a script that orchestrates the tools. The script runs outside the agent’s context, processes all the data, and returns only the final result.

User: "Generate a report of all customers who haven't ordered in 90 days
       and check if they have open support tickets."

Agent: "I'll write a script to analyze this."

Agent calls: execute_workflow(
  code: """
    from datetime import datetime, timedelta

    ninety_days_ago = datetime.now() - timedelta(days=90)
    customers = await tools.get_all_customers()

    inactive_with_tickets = []
    for customer in customers:
        orders = await tools.get_orders(customer_id=customer["id"])
        last_order_date = max((o["date"] for o in orders), default=None) if orders else None

        if last_order_date is None or last_order_date < ninety_days_ago:
            tickets = await tools.get_support_tickets(
                customer_id=customer["id"],
                status="open"
            )
            if tickets:
                inactive_with_tickets.append({
                    "name": customer["name"],
                    "email": customer["email"],
                    "last_order": last_order_date.strftime("%b %d, %Y") if last_order_date else "Never",
                    "open_tickets": len(tickets)
                })

    return {
        "count": len(inactive_with_tickets),
        "customers": inactive_with_tickets[:20],
        "has_more": len(inactive_with_tickets) > 20
    }
  """
)

Tool returns: {
  count: 47,
  customers: [
    { name: "Acme Corp", email: "buyer@acme.com", last_order: "Oct 15, 2024",
      open_tickets: 3 },
    ...
  ],
  has_more: true
}

Agent: "I found 47 customers who haven't ordered in 90 days and have open support
        tickets. Here are the first 20..."

The agent never saw the 2,847 customer records. It never examined each order history. It never processed the support ticket queries one by one. All of that happened inside the script, invisible to the agent. The agent only saw the final summary: 47 customers with the details it needs to respond to the user.

This is Scripted Orchestration. Instead of the agent manually stepping through each operation and looking at each result, it writes code that does the stepping. The code is deterministic, fast, and cheap. The agent just sees the output.
Three Big Advantages

Scripted Orchestration has three major advantages over step-by-step tool calling:

It’s dramatically more token-efficient. In the step-by-step approach, all 2,847 customer records flow through the agent’s context, plus thousands of order history responses, plus support ticket queries. That could easily be 500,000+ tokens. With Scripted Orchestration, the agent sees maybe 500 tokens—just the final summary. That’s a 1000x difference.

It’s deterministic. The script does exactly what it says, every time. There’s no variation based on how the agent interprets intermediate results, no risk of the agent getting confused halfway through and going down the wrong path. The logic is explicit in the code.

It’s faster. Each step-by-step tool call requires a round trip through the agent—the tool returns, the agent reasons about it, the agent decides what to do next, the agent makes another call. With Scripted Orchestration, the tools are called directly from the script without waiting for agent reasoning between each call. For a workflow with thousands of operations, this can be the difference between minutes and hours.
Implementing the Execute Workflow Tool

To enable Scripted Orchestration, you need a tool that can execute agent-written code. Here’s how to build one.

First, set up the basic structure:

from mcp.server.fastmcp import FastMCP
from pydantic import Field

mcp = FastMCP("workflow-server")

Next, define the tools that will be available inside the script. These are your existing tools, wrapped so they can be called from the workflow code:

class WorkflowTools:
    async def get_all_customers(self):
        return await db.query("SELECT * FROM customers")

    async def get_orders(self, customer_id: str):
        return await db.query(
            "SELECT * FROM orders WHERE customer_id = $1",
            [customer_id]
        )

    async def get_support_tickets(self, customer_id: str, status: str = None):
        if status:
            return await db.query(
                "SELECT * FROM tickets WHERE customer_id = $1 AND status = $2",
                [customer_id, status]
            )
        return await db.query(
            "SELECT * FROM tickets WHERE customer_id = $1",
            [customer_id]
        )

workflow_tools = WorkflowTools()

Now create the execution environment. This is a restricted Python context that has access to the workflow tools but is isolated from the rest of the system:

@mcp.tool()
async def execute_workflow(
    code: str = Field(description="Python code to execute. Has access to 'tools' object.")
) -> str:
    # Create a restricted global namespace
    restricted_globals = {
        "__builtins__": {
            # Safe built-ins only
            "len": len,
            "sum": sum,
            "min": min,
            "max": max,
            "sorted": sorted,
            "list": list,
            "dict": dict,
            "str": str,
            "int": int,
            "float": float,
            "range": range,
            "True": True,
            "False": False,
            "None": None,
        },
        "tools": workflow_tools,
    }

    # Wrap the code in an async function
    wrapped_code = f"""
async def __workflow__():
{chr(10).join('    ' + line for line in code.split(chr(10)))}

__result__ = __workflow__()
"""

    try:
        # Compile and execute
        compiled = compile(wrapped_code, "<workflow>", "exec")
        local_namespace = {}
        exec(compiled, restricted_globals, local_namespace)

        # Get the coroutine and run it with timeout
        coro = local_namespace["__result__"]
        result = await asyncio.wait_for(coro, timeout=60.0)

        return tool_response({"status": "success", "result": result})

    except asyncio.TimeoutError:
        return tool_response({"status": "error", "error": "Workflow timed out"})
    except Exception as e:
        return tool_response({"status": "error", "error": str(e)})

The sandbox is important for security. The workflow code can access the registered tools, but it can’t access the file system, make arbitrary network requests, or interfere with the running process. The timeout prevents infinite loops.
How the Agent Uses It

When the agent encounters a task that involves iterating through data, combining information from multiple sources, or applying logic to filter results, it can write a script instead of doing each step manually.

Here’s another example—analyzing which enterprise customers have the highest support ticket rate per dollar spent:

User: "Which of our enterprise customers have the highest support ticket rate
       per dollar spent?"

Agent calls: execute_workflow(
  code: """
    customers = await tools.get_all_customers()
    enterprise = [c for c in customers if c["tier"] == "enterprise"]

    analysis = []
    for customer in enterprise:
        orders = await tools.get_orders(customer_id=customer["id"])
        tickets = await tools.get_support_tickets(customer_id=customer["id"])

        total_spend = sum(o["amount"] for o in orders)
        ticket_rate = (len(tickets) / total_spend) * 1000 if total_spend > 0 else 0

        analysis.append({
            "name": customer["name"],
            "total_spend": total_spend,
            "ticket_count": len(tickets),
            "tickets_per_1k": round(ticket_rate, 2)
        })

    analysis.sort(key=lambda a: a["tickets_per_1k"], reverse=True)
    avg_rate = sum(a["tickets_per_1k"] for a in analysis) / len(analysis) if analysis else 0

    return {
        "count": len(analysis),
        "highest_rate": analysis[:10],
        "lowest_rate": list(reversed(analysis[-5:])),
        "average_rate": round(avg_rate, 2)
    }
  """
)

The agent is expressing what it wants—find enterprise customers, calculate ticket rates, sort and summarize—without manually stepping through each customer. The script handles the iteration, the data fetching, the calculations, and the summarization. The agent just gets back the answer.

Notice how the script can use Python’s asyncio to fetch data efficiently. This is something the agent couldn’t easily do with step-by-step tool calls—it would have to call one tool, wait for the result, call another tool, wait for that result. The script can express these operations naturally.
Pre-Built vs. Ad-Hoc

You don’t always need the agent to write custom scripts. For common workflows, you can pre-build them as dedicated tools:

@mcp.tool()
async def analyze_customer_retention(
    days_inactive: int = Field(default=90, description="Days to consider inactive"),
    tier: Literal["all", "enterprise", "standard"] = Field(default="all"),
    include_ticket_info: bool = Field(default=True)
) -> str:
    cutoff_date = datetime.now() - timedelta(days=days_inactive)

    customers = await db.query("SELECT * FROM customers")
    if tier != "all":
        customers = [c for c in customers if c["tier"] == tier]

    analysis = []
    for customer in customers:
        orders = await db.query(
            "SELECT * FROM orders WHERE customer_id = $1",
            [customer["id"]]
        )
        last_order = max((o["date"] for o in orders), default=None) if orders else None

        if last_order is None or last_order < cutoff_date:
            result = {
                "name": customer["name"],
                "last_order": last_order.strftime("%b %d, %Y") if last_order else "Never",
                "total_orders": len(orders)
            }

            if include_ticket_info:
                tickets = await db.query(
                    "SELECT * FROM tickets WHERE customer_id = $1 AND status = 'open'",
                    [customer["id"]]
                )
                result["open_tickets"] = len(tickets)

            analysis.append(result)

    return tool_response({
        "inactive_count": len(analysis),
        "customers": analysis[:20],
        "has_more": len(analysis) > 20
    })

Pre-built tools are simpler for the agent to use—just call the tool with parameters. But they’re less flexible. The agent can only do what the tool was designed to do.

Scripted Orchestration is for when you need flexibility. When the user asks something novel that your pre-built tools don’t cover, the agent can compose a custom workflow on the spot. In practice, you’ll probably have both: pre-built tools for common operations, and execute_workflow for edge cases and novel combinations.
Security Considerations

Scripted Orchestration involves executing agent-generated code. This requires careful security design.

Sandbox the execution environment. The workflow code should only have access to the tools you explicitly provide. Block access to the file system, network (except through your tools), process management, and code execution APIs like eval or exec. The example above shows how to do this by restricting the __builtins__ namespace in Python.

Enforce timeouts. Agent-generated code could contain infinite loops, either by accident or because the agent made a mistake. Always set a maximum execution time. 60 seconds is usually plenty; most workflows complete in under a second.

Limit resource usage. Consider tracking how many tool calls a workflow makes. A workflow that tries to make a million database queries is probably buggy or malicious. You might cap it at 10,000 operations, or implement rate limiting within the workflow.

Log everything. Keep records of what workflows were executed and what code they contained. This is essential for debugging when something goes wrong, and for security auditing if you’re worried about misuse.

The security constraints you need depend on your deployment context. A coding assistant running locally has different requirements than a customer-facing agent processing financial data.
When to Use Scripted Orchestration

Scripted Orchestration is most valuable when:

The workflow involves iterating through lots of data. If you need to check something for each of 1,000 customers, that’s 1,000 tool calls in the step-by-step approach. A script does it in one.

The intermediate data isn’t interesting. If the agent doesn’t need to reason about each individual customer record—it’s just checking conditions and filtering—then there’s no point in putting all that data into the context.

The logic is expressible in code. Filters, calculations, aggregations, joins between data sources—these are all things that code does well. If the task is primarily mechanical, let code handle it.

Speed matters. Thousands of round-trips through the agent take time. A script runs through them much faster.

Scripted Orchestration is less appropriate when:

The agent needs to make decisions at each step. If the right action depends on interpreting something semantically—“does this customer seem unhappy?”—the agent might need to see each case. (Though you could also handle this with self-prompting in the script—tools can make their own LLM calls with isolated, focused prompts.)

Transparency matters. Sometimes the user wants to see each step happening. Sometimes you need the agent to explain its reasoning along the way. Scripted Orchestration happens invisibly; if visibility is important, step-by-step might be better.

The workflow is simple. If you’re just calling one or two tools, writing a script is overkill. Just call the tools directly.
Key Takeaways

Scripted Orchestration shifts the agent’s role from executing each step to specifying the logic. The agent writes code that describes what should happen; the code runs outside the context window.

It’s way more token-efficient. Intermediate data never touches the agent’s context. The agent sees only the final result.

It’s deterministic. The script does exactly what it says. No variation based on agent interpretation.

It’s faster. No round-trips through the agent between each tool call.

It enables complex workflows. Parallel operations, nested loops, multi-source joins—things that are awkward or impossible with step-by-step tool calling become natural in code.

Security requires attention. Sandbox the environment, enforce timeouts, limit resources, and log everything.

When your agent is doing mechanical work—iterating, filtering, aggregating—consider whether that work could be expressed as a script. The agent is good at understanding what the user wants and communicating results. Let code handle the mechanical parts.
