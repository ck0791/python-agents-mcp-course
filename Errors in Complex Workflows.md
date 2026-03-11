
## Failing Forward, Part 2: Building Error Chains

In the previous tutorial, we built an expense submission tool where each error teaches the agent what to do next. Now we’ll tackle something harder: multi-step processes where the agent can’t possibly know the full workflow in advance.
The Problem: Processes Too Big for Context

Imagine you’re building an expense system for a company with 10,000 employees. The expense policies fill a 50-page document. There are different rules for different departments, spending limits that vary by role, special approval workflows for travel vs. software vs. meals, and exceptions for conferences, client entertainment, and emergencies.

No agent can hold all of this in context. And even if it could, the policies change quarterly.

This is where error chains become essential. Instead of front-loading every rule into the agent’s system prompt, we let the tools teach the process as the agent encounters each step. The agent doesn’t need to know that late expenses require manager approval—until it tries to submit a late expense. Then the error tells it exactly what to do.

Think of it like a GPS. You don’t memorize every turn before you start driving. You follow the next instruction, and when something changes (road closed, traffic ahead), the GPS recalculates and gives you a new next step. Error chains work the same way.
Exercise Solution: Date Validation

Let’s add date validation to our expense tool. We need two rules:

    Future dates are rejected (probably a typo)
    Expenses older than 90 days need manager approval

The key insight: when we detect an old expense, we don’t just reject it. We tell the agent how to get it approved.

Here’s the old expense check. Look at what information we pack into the error:

days_since_expense = (today - expense_date).days

if days_since_expense > LATE_EXPENSE_DAYS and not approval_id:
    return tool_response(
        ToolResult(
            status="needs_action",
            error="late_expense",
            message=f"This expense is {days_since_expense} days old. Expenses older than {LATE_EXPENSE_DAYS} days require manager approval.",
            next_action="request_late_expense_approval",
            next_action_params={
                "expense_date": date,
                "days_late": days_since_expense - LATE_EXPENSE_DAYS,
                "amount": amount,
                "category": category,
                "description": description,
            },
            hint="This expense is quite old. Tell the user it needs manager approval first.",
        )
    )

Notice what’s happening here:

    We name the next tool to call: request_late_expense_approval
    We pre-fill the parameters: The agent doesn’t need to figure out what data to pass—we hand it everything
    We explain the situation: The hint tells the agent how to communicate with the user

The agent didn’t know this approval workflow existed until it hit this error. Now it knows exactly what to do. It doesn’t need to have read the 50-page policy document. The tool taught it just-in-time.
How the Chain Unfolds

Let’s trace through what happens when a user says “Submit my $150 dinner from last December”:

Agent calls: submit_expense(amount: 150, category: "meals", date: "2024-12-15")

Tool returns:
  error: "expense_too_old"
  next_action: "request_late_expense_approval"
  next_action_params: { expense_date: "2024-12-15", expense_amount: 150, ... }

The agent now knows: “I need to call request_late_expense_approval with these parameters.” It doesn’t need to reason about company policy. It just follows the instruction.

Agent calls: request_late_expense_approval(expense_date: "2024-12-15", expense_amount: 150, ...)

Tool returns:
  status: "success"
  approval_id: "apr_123"
  approval_status: "pending"
  next_action: "wait_for_approval"
  hint: "Tell the user their manager has been notified."

Now the agent knows to wait. It tells the user: “Your manager needs to approve this late expense first.”

Later, when the user comes back:

Agent calls: submit_expense(amount: 150, date: "2024-12-15", late_approval_id: "apr_123")

Tool returns:
  error: "receipt_required"
  next_action: "upload_receipt"

A different error! But the agent handles it the same way—follow the next instruction.

This is the power of error chains: complex multi-step workflows emerge from simple local decisions. Each tool only needs to know about its immediate next step, not the entire process.
Visualizing the Chain

Here’s the full flow as a diagram:

submit_expense (failed: expense_too_old)
       │
       ▼
request_late_expense_approval → success (pending)
       │
       ▼
       ... time passes, manager approves ...
       │
       ▼
submit_expense (failed: receipt_required)
       │
       ▼
upload_receipt → success (receipt_url)
       │
       ▼
submit_expense → success!

Each error is a signpost pointing to the next step. The agent never has to figure out the entire workflow—it just follows the breadcrumbs.
Building the Approval Tool

Now we need to implement request_late_expense_approval—the tool that the error told the agent to call. This tool also follows the pattern: even its success response guides the agent on what to do next.

Here’s the key part—the success response:

return tool_response(
    ToolResult(
        status="success",
        message=f"Approval request sent to {approver.name}",
        next_action="wait_for_approval",
        hint="Tell the user their manager has been notified. They can check status using check_approval_status.",
        extra={
            "approval_id": approval.id,
            "approval_status": "pending",
            "user_message": f"I've sent an approval request to your manager ({approver.name}). This usually takes 1-2 business days.",
        },
    )
)

Even success responses guide the agent:

    next_action: “wait_for_approval” (don’t keep trying to submit)
    hint: How to communicate with the user
    user_message: Ready-to-use text for the response

The agent doesn’t have to figure out what to tell the user. The tool provides a suggested message.
Ordering Your Validations

When a single request might fail multiple validations, the order matters. Consider an expense that’s:

    120 days old (needs approval)
    Missing a receipt (needs upload)
    Has an invalid category (needs fix)

Which error should we show first?

Show quick fixes before slow workflows. If the date is wrong (typo), tell them immediately—don’t make them wait for manager approval only to discover they typed “2025” instead of “2024”.

Here’s a good ordering principle:

    Format errors — Can’t proceed without valid data
    Quick user fixes — Future date? Probably a typo
    Slow workflows — Manager approval takes days, do it last
    Agent-fixable issues — Invalid category? Agent can map “office supplies” → “supplies”

This way, users fix the easy stuff first. They don’t start a multi-day approval process only to hit another error afterward.
Exercise: Build the Approval Status Tool

The request_late_expense_approval tool tells the agent to use check_approval_status to monitor the approval. Implement this tool with failing-forward errors for these cases:

    Approval not found: The ID doesn’t exist
    Approval denied: The manager rejected it
    Approval expired: Approvals are only valid for 30 days

For each case, include appropriate next actions:

    Not found: Maybe suggest searching by date range?
    Denied: Include the denial reason and who to contact
    Expired: Suggest creating a new approval request

Think about what information the agent would need to help the user in each scenario.

The solution is in the next tutorial.
