
## Failing Forward, Part 1: Errors as Curriculum

Something Fundamental Has Changed

For decades, we’ve built APIs and functions for callers that can’t adapt. When a traditional program hits an error, that’s it—the code path was fixed at compile time. The caller can’t decide to try a different approach, gather more information, or ask clarifying questions. It does exactly what it was programmed to do, and if that doesn’t work, it fails.

Because our callers were inflexible, we never bothered making error messages instructive. What’s the point of telling a program how to fix a problem when it can’t change its behavior anyway? The best we could do was log something useful for a human to debug later. The error message was never meant for the caller—it was meant for a developer who might read it days or weeks after the fact.

Agents change this completely.

An agent can read an error message and actually do something about it. It can change what tools it calls, in what sequence, with what parameters—all on the fly, mid-conversation. No recompilation. No redeployment. No waiting for a human to fix the code. The agent adapts in real time.

This means error messages are no longer just diagnostic artifacts. They’re instructions. For the first time, the thing calling your API can understand natural language, reason about what went wrong, and choose a different approach. We can finally take advantage of this intelligence.

We call this pattern Failing Forward: designing errors that teach the agent how to succeed, right when it needs to learn.
The Opportunity We’ve Been Missing

Think about what this unlocks. Instead of:

Error: Receipt required

We can return:

This expense is $47.50, which exceeds the $25 threshold for receipts.
Call upload_receipt with formats: [jpg, png, pdf], max size: 10MB.
Then retry submit_expense with the returned receipt_url.

A traditional caller couldn’t do anything with that second message. But an agent? It reads it, understands exactly what to do, and does it. The error becomes a course correction, not a dead end.

This is the mental shift: error messages are now runtime instructions for an intelligent system. The quality of those instructions directly determines whether your agent succeeds or fails.

In this tutorial series, we’ll explore patterns and techniques to fail forward—so your agents learn on the fly, adapt to problems as they encounter them, and don’t get stuck in loops of failure wasting tokens and potentially wreaking havoc.
The Typical Approach (And Why It Fails)

Let’s build an expense submission tool for an MCP server. We’ll start with the typical approach, then transform it into a teaching tool.

Here’s what most developers write:

from mcp.server.fastmcp import FastMCP

server = FastMCP("expense-server")

VALID_CATEGORIES = ["meals", "travel", "supplies", "software"]

@server.tool()
async def submit_expense(
    amount: float,
    category: str,
    description: str,
    receipt_url: str | None = None,
) -> str:
    """Submit an expense for reimbursement."""

    # Validation that doesn't teach
    if amount <= 0:
        raise ValueError("Invalid amount")

    if amount > 25 and not receipt_url:
        raise ValueError("Receipt required")

    if category not in VALID_CATEGORIES:
        raise ValueError("Invalid category")

    # Submit the expense...
    expense = await database.create(amount, category, description, receipt_url)

    return f"Expense {expense.id} submitted"

When this tool fails, here’s what the agent sees:

Error: Receipt required

The agent now has to guess:

    What kind of receipt?
    What format?
    Where should it upload?
    What’s the size limit?

It might retry without a receipt, try a different format, or just tell the user “something went wrong.”

This is the failure mode we need to eliminate.
The Failing Forward Version

Now let’s rewrite this tool so every error becomes a learning opportunity. The key is to answer three questions in every error response:

    What went wrong? (so the agent understands the situation)
    What should it do next? (so it doesn’t have to guess)
    What parameters does it need? (so it can act immediately)

Here’s the transformed tool:

import json
from dataclasses import dataclass, field
from typing import Any, Literal

from mcp.server.fastmcp import FastMCP

server = FastMCP("expense-server")

VALID_CATEGORIES = ["meals", "travel", "supplies", "software"]


# Define a structured result type
@dataclass
class ToolResult:
    status: Literal["success", "failed", "needs_clarification"]
    message: str
    error: str | None = None
    next_action: str | None = None
    next_action_params: dict[str, Any] | None = None
    hint: str | None = None
    extra: dict[str, Any] = field(default_factory=dict)

    def to_dict(self) -> dict[str, Any]:
        result: dict[str, Any] = {"status": self.status, "message": self.message}
        if self.error:
            result["error"] = self.error
        if self.next_action:
            result["next_action"] = self.next_action
        if self.next_action_params:
            result["next_action_params"] = self.next_action_params
        if self.hint:
            result["hint"] = self.hint
        result.update(self.extra)
        return result


def tool_response(result: ToolResult) -> str:
    return json.dumps(result.to_dict(), indent=2)


@server.tool()
async def submit_expense(
    amount: float,
    category: str,
    description: str,
    receipt_url: str | None = None,
) -> str:
    """
    Submit an expense for reimbursement.

    Args:
        amount: Expense amount in USD
        category: One of: meals, travel, supplies, software
        description: Brief description of the expense
        receipt_url: URL to receipt image (required for expenses over $25)
    """

    # Teach about amount validation
    if amount <= 0:
        return tool_response(
            ToolResult(
                status="failed",
                error="invalid_amount",
                message="Expense amount must be greater than zero",
                next_action="resubmit_expense",
                hint="Check that the amount is positive. The user may have entered it incorrectly.",
            )
        )

    # Teach about receipt requirements
    if amount > 25 and not receipt_url:
        return tool_response(
            ToolResult(
                status="failed",
                error="receipt_required",
                message=f"Expenses over $25 require a receipt. This expense is ${amount:.2f}.",
                next_action="upload_receipt",
                next_action_params={
                    "expense_amount": amount,
                    "supported_formats": ["image/jpeg", "image/png", "application/pdf"],
                    "max_size_mb": 10,
                    "upload_endpoint": "/api/receipts/upload",
                },
                hint="Ask the user to provide a photo or scan of the receipt. Once uploaded, retry submit_expense with the receipt_url.",
            )
        )

    # Teach about valid categories
    if category not in VALID_CATEGORIES:
        return tool_response(
            ToolResult(
                status="failed",
                error="invalid_category",
                message=f"Category '{category}' is not recognized.",
                next_action="resubmit_expense",
                hint=f"The user said '{category}'. Map this to one of the valid categories: {', '.join(VALID_CATEGORIES)}",
                extra={"valid_options": VALID_CATEGORIES},
            )
        )

    # Success - submit the expense
    expense = await database.create(amount, category, description, receipt_url)

    return tool_response(
        ToolResult(
            status="success",
            message=f"Expense submitted for ${amount:.2f} in {category}",
            extra={
                "expense_id": expense.id,
                "current_status": "pending_approval",
                "user_message": f"Your expense for ${amount:.2f} has been submitted and is pending approval.",
            },
        )
    )

Now when the receipt validation fails, the agent sees:

{
  "status": "failed",
  "error": "receipt_required",
  "message": "Expenses over $25 require a receipt. This expense is $47.50.",
  "next_action": "upload_receipt",
  "next_action_params": {
    "expense_amount": 47.50,
    "supported_formats": ["image/jpeg", "image/png", "application/pdf"],
    "max_size_mb": 10,
    "upload_endpoint": "/api/receipts/upload"
  },
  "hint": "Ask the user to provide a photo or scan of the receipt. Once uploaded, retry submit_expense with the receipt_url."
}

The agent knows:

    What failed: receipt_required
    Why it failed: Expenses over $25 require receipts, and this one is $47.50
    What to do next: Call upload_receipt
    How to do it: Use these specific formats, this size limit, this endpoint
    What to tell the user: Ask for a photo or scan

No guessing. No wasted tokens. The error message is a complete instruction manual.
The Structure of a Teaching Error

Every failing-forward error has these components:

{
    # What happened
    "status": "failed",
    "error": "error_code",           # Machine-readable error type
    "message": "Human explanation",  # What went wrong

    # What to do about it
    "next_action": "tool_name",      # The tool to call next
    "next_action_params": {...},     # Parameters for that tool

    # Context for the agent
    "hint": "Strategy suggestion"    # How to communicate with the user
}

The next_action field is particularly powerful. It creates a directed graph of tool calls:

submit_expense (failed: receipt_required)
       │
       ▼
  upload_receipt (returns receipt_url)
       │
       ▼
submit_expense (with receipt_url) → success

The agent doesn’t need to reason about what to do next. The error tells it explicitly.
Handling Category Mismatches

The category validation shows another important pattern. When the user says something like “office supplies” but your system expects “supplies”, the error should help the agent make the mapping:

if category not in VALID_CATEGORIES:
    return tool_response(
        ToolResult(
            status="failed",
            error="invalid_category",
            message=f"Category '{category}' is not recognized.",
            next_action="resubmit_expense",
            hint=f"Map '{category}' to one of the valid categories: {', '.join(VALID_CATEGORIES)}",
            extra={"valid_options": VALID_CATEGORIES},
        )
    )

The hint tells the agent: “You have what was passed, you have the valid options, figure out the best match.” This is the right division of labor—the tool enforces valid values, the agent handles the fuzzy mapping from natural language.
Testing Error Messages

When building MCP tools, test your error messages the same way you test your success paths. For each validation:

    Trigger the error intentionally
    Read the error message as if you were the agent
    Ask: “Do I know exactly what to do next?”

If the answer is no, your error isn’t teaching—it’s just complaining.

# Test: submit expense without receipt for amount > $25
result = await session.call_tool(
    name="submit_expense",
    arguments={
        "amount": 50.00,
        "category": "meals",
        "description": "Team lunch",
        # Intentionally omitting receipt_url
    },
)

# Verify the error teaches
response = json.loads(get_result_text(result))
assert response["status"] == "failed"
assert response["error"] == "receipt_required"
assert response["next_action"] == "upload_receipt"
assert "image/jpeg" in response["next_action_params"]["supported_formats"]

Exercise: Add Date Validation

The expense tool currently doesn’t validate dates. Add validation for these cases:

    Future dates: Expenses can’t be dated in the future
    Old expenses: Expenses older than 90 days need special approval

For each case, return a failing-forward error that teaches the agent what to do. Include:

    The error code
    A clear message
    The next action (or how to get special approval)
    A hint for communicating with the user

Think about what parameters the agent would need. For old expenses, it might need to know how old the expense is and who can grant approval.
What’s Next

This tutorial introduced the core idea: errors should teach, not just complain. But we’ve only scratched the surface.

In the next tutorials, we’ll explore:

    Error Chains: What happens when fixing one error reveals another? How do you guide an agent through multi-step recovery processes?
    Pre-filled Parameters: How to hand the agent everything it needs so it can act immediately without additional reasoning
    Validation Ordering: When multiple things are wrong, which error do you show first?
    Alternative Actions: What if there’s more than one way to recover?

The goal is to build tools that make agent failure nearly impossible—not by preventing errors, but by making recovery automatic.

The solution to the exercise, and the next pattern, is in Part 2.
