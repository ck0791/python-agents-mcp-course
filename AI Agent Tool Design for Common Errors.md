
## Intelligence Budget: Validate at Source

The Validation Reflex

When a tool receives input, the first instinct is often to validate it minimally and let the agent handle edge cases. After all, the agent can understand context. The agent can make judgment calls. The agent can ask clarifying questions.

This reflex is expensive.

Consider what happens when a user submits an expense with the description “asdfasdf”. A minimal tool just accepts it:

User: "Expense for $500, description: asdfasdf"

Agent calls: submit_expense(amount: 500, description: "asdfasdf")

Tool returns: {
  status: "success",
  expense_id: "exp_123"
}

Agent: "I've submitted your expense of $500."

// One week later, during audit...
Auditor: "What is 'asdfasdf'? This expense is rejected."

The problem here isn’t just that a garbage description got through. The problem is that the tool missed an opportunity to catch an obvious issue and either fix it or flag it immediately.

But the alternative—sending the description back to the agent for validation—has its own costs:

User: "Expense for $500, description: asdfasdf"

Agent calls: submit_expense(amount: 500, description: "asdfasdf")

Tool returns: {
  status: "created",
  expense: {
    id: "exp_123",
    amount: 500,
    description: "asdfasdf",
    // ... more fields
  }
}

Agent reasoning: "Let me check if this description is valid...
'asdfasdf' doesn't seem like a real expense description...
It looks like random characters...
I should ask the user for clarification..."

Agent: "The description 'asdfasdf' doesn't seem meaningful. Could you provide more details about this expense?"

Now the agent is doing validation work—examining the description, reasoning about whether it’s acceptable, formulating a response. All of this consumes attention budget. And worse, the expense data had to flow into the agent’s context just so the agent could examine it and flag a problem.

There’s a better way: validate at source.
The Validate at Source Principle

Validation should happen where the data lives—inside the tool—not after routing data to the agent for inspection.

This applies to all forms of validation:

    Format checks (is this a valid email address?)
    Business rules (is this amount within limits?)
    Semantic checks (does this description make sense?)

When validation happens in the tool, the agent only sees results: either the operation succeeded, or it failed with a clear explanation of what went wrong. The agent never needs to inspect the data itself.

The principle extends to semantic validation too. Just because validating “asdfasdf” requires understanding that it’s gibberish doesn’t mean the agent needs to do that check. A self-prompted LLM call in the tool can make that judgment, keeping the reasoning isolated and the agent’s context clean.
Building the Validation Stack

Think of validation as a stack of increasingly expensive checks. Each layer filters out invalid inputs before reaching the next layer:

┌─────────────────────────────────────────────────────────────┐
│  LAYER 1: Format & Type Validation                          │
│  • Schema validation (Pydantic, JSON Schema)                │
│  • Type coercion and parsing                                │
│  • Required field checks                                    │
│                                                             │
│  Cost: Zero                                                 │
│  Speed: Microseconds                                        │
└─────────────────────────────────────────────────────────────┘
                              │
            (Only valid formats reach here)
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 2: Business Rule Validation                          │
│  • Amount limits and thresholds                             │
│  • Date range checks                                        │
│  • Category constraints                                     │
│  • Cross-field consistency                                  │
│                                                             │
│  Cost: Zero (pure code)                                     │
│  Speed: Microseconds                                        │
└─────────────────────────────────────────────────────────────┘
                              │
            (Only business-valid inputs reach here)
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 3: Semantic Validation (Self-Prompting)              │
│  • Is this description meaningful?                          │
│  • Does this make sense for this category?                  │
│  • Are there policy red flags?                              │
│                                                             │
│  Cost: ~100-200 tokens (self-prompted LLM call)             │
│  Speed: 500ms-2s                                            │
└─────────────────────────────────────────────────────────────┘
                              │
            (Only semantically valid inputs reach here)
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 4: Human Review (if still uncertain)                 │
│  • Low-confidence cases flagged for review                  │
│  • High-stakes decisions queued for approval                │
│                                                             │
│  Cost: Human attention                                      │
│  Speed: Minutes to days                                     │
└─────────────────────────────────────────────────────────────┘

Most inputs never reach Layer 3. Format errors are caught instantly. Business rule violations are caught by code. Only genuinely ambiguous cases—where semantic judgment is needed—incur the cost of an LLM call.

And even when semantic validation is needed, it happens in the tool through self-prompting. The agent never sees the problematic data; it only sees the validation result.
Implementing the Stack

Let’s build out each layer for an expense submission tool.
Layer 1: Format Validation

This happens automatically through your schema definition, but you can add explicit checks too:

from pydantic import Field
from typing import Optional, Literal

@mcp.tool()
async def submit_expense(
    amount: float = Field(gt=0, description="Amount must be positive"),
    description: str = Field(min_length=3, max_length=500, description="Description required"),
    category: Literal["meals", "travel", "supplies", "software", "client_entertainment", "team_meals"] = Field(
        description="Expense category"
    ),
    date: Optional[str] = Field(default=None, description="Date must be YYYY-MM-DD format"),
    receipt_url: Optional[str] = Field(default=None, description="URL to receipt"),
) -> str:
    # If we get here, basic format validation already passed
    # ...

Pydantic handles most format validation automatically. Invalid inputs never even reach your handler—they fail with a clear error message that the agent can relay to the user.
Layer 2: Business Rules

These are your company-specific constraints implemented as code:

from datetime import datetime, timedelta
from typing import Dict, Any, Optional

async def validate_business_rules(
    amount: float,
    category: str,
    date: Optional[str] = None
) -> Dict[str, Any]:
    """Validate business rules for an expense."""

    # Amount limits by category
    category_limits = {
        "meals": 100,
        "team_meals": 75,  # per person estimate
        "client_entertainment": 300,
        "travel": 5000,
        "supplies": 500,
        "software": 1000
    }

    max_amount = category_limits.get(category, 500)
    if amount > max_amount:
        return {
            "valid": False,
            "error": f"{category} expenses are limited to ${max_amount}. This expense of ${amount} exceeds that limit."
        }

    # Date validation (if provided)
    if date:
        try:
            expense_date = datetime.fromisoformat(date)
            now = datetime.now()
            ninety_days_ago = now - timedelta(days=90)

            if expense_date > now:
                return {
                    "valid": False,
                    "error": "Expense date cannot be in the future."
                }

            if expense_date < ninety_days_ago:
                return {
                    "valid": False,
                    "error": "Expenses older than 90 days cannot be submitted. Please contact finance for exceptions."
                }

            # Weekend client entertainment check
            if category == "client_entertainment":
                day_of_week = expense_date.weekday()
                if day_of_week in (5, 6):  # Saturday = 5, Sunday = 6
                    # Weekend client entertainment requires justification
                    # We'll handle this in semantic validation
                    return {
                        "valid": True,
                        "warning": "weekend_client_entertainment"
                    }
        except ValueError:
            return {
                "valid": False,
                "error": "Invalid date format. Use YYYY-MM-DD."
            }

    return {"valid": True}

These checks are instant and free. They catch the majority of invalid submissions without any LLM involvement.
Layer 3: Semantic Validation

This is where self-prompting comes in. For cases that require understanding:

import re
import json
from openai import OpenAI
from typing import Dict, Any, Optional

openai_client = OpenAI()

async def validate_semantics(
    description: str,
    category: str,
    amount: float,
    flags: Optional[Dict[str, bool]] = None
) -> Dict[str, Any]:
    """
    Semantic validation uses SELF-PROMPTING to determine if the expense
    description is meaningful and appropriate.
    """
    # Quick deterministic checks for obvious problems
    # These don't need LLM

    # Check for gibberish patterns
    gibberish_patterns = [
        r'^[a-z]{5,}$',   # Random letters like "asdfgh"
        r'^[\d\s]+$',      # Just numbers and spaces
        r'(.)\1{4,}',      # Repeated characters like "aaaaa"
        r'^test',          # Test entries
    ]

    description_stripped = description.strip()
    for pattern in gibberish_patterns:
        if re.match(pattern, description_stripped, re.IGNORECASE):
            return {
                "valid": False,
                "confidence": 0.95,
                "issues": ["Description appears to be placeholder or test text"],
                "suggestions": ["Provide a meaningful description of what this expense was for"]
            }

    # Check minimum meaningful content
    words = description_stripped.split()
    if len(words) < 2:
        return {
            "valid": False,
            "confidence": 0.90,
            "issues": ["Description is too brief for audit purposes"],
            "suggestions": ["Include what the expense was for and any relevant context"]
        }

    # For genuinely ambiguous cases, use self-prompting
    weekend_note = ""
    if flags and flags.get("weekend_entertainment"):
        weekend_note = "Note: This is weekend client entertainment."

    response = openai_client.responses.create(
        model="gpt-4o-mini",
        input=[
            {
                "role": "user",
                "content": f"""Category: {category}
Amount: ${amount}
Description: "{description}"
{weekend_note}""",
            }
        ],
        instructions="""You validate expense report descriptions. Check for:

1. MEANINGFULNESS: Is this a real expense description or placeholder text?
2. CATEGORY MATCH: Does the description match the stated category?
   - meals: Food purchases for individual work meals
   - client_entertainment: Meals/events with clients, customers, prospects
   - team_meals: Team lunches, celebrations, department events
   - travel: Transportation, lodging, related expenses
   - supplies: Office supplies and equipment
   - software: Software subscriptions and licenses

3. REASONABLENESS: Is the amount reasonable for what's described?
   - $500 coffee -> suspicious
   - $15 lunch -> reasonable
   - $200 team dinner for 10 people -> reasonable

4. POLICY FLAGS: Any concerns?
   - Mentions of alcohol in large amounts
   - Luxury items not clearly justified
   - Vague descriptions for large amounts

Respond with JSON only:
{
  "valid": true/false,
  "confidence": 0.0-1.0,
  "issues": ["list of problems, empty array if valid"],
  "suggestions": ["how to fix, empty array if valid"]
}""",
        temperature=0,
    )

    text = response.output_text or ""
    try:
        json_match = re.search(r'\{[\s\S]*\}', text)
        if json_match:
            return json.loads(json_match.group())
    except Exception:
        pass

    return {"valid": True, "confidence": 0.6}

Notice the layering even within semantic validation. Obvious gibberish is caught with regex—no LLM needed. Only genuinely ambiguous cases trigger the self-prompted LLM call.
Putting It Together

Here’s the complete tool with all validation layers:

@mcp.tool()
async def submit_expense(
    amount: float = Field(gt=0, description="Amount must be positive"),
    description: str = Field(min_length=1, max_length=500, description="Description required"),
    category: Literal["meals", "travel", "supplies", "software", "client_entertainment", "team_meals"] = Field(
        description="Expense category"
    ),
    date: Optional[str] = Field(default=None, description="Date must be YYYY-MM-DD"),
    receipt_url: Optional[str] = Field(default=None, description="URL to receipt"),
) -> str:
    # Layer 2: Business rules
    business_check = await validate_business_rules(amount, category, date)

    if not business_check["valid"]:
        return tool_response({
            "status": "rejected",
            "validation_layer": "business_rules",
            "tell_user": business_check["error"]
        })

    # Layer 3: Semantic validation
    semantic_check = await validate_semantics(
        description,
        category,
        amount,
        {"weekend_entertainment": business_check.get("warning") == "weekend_client_entertainment"}
    )

    if not semantic_check["valid"]:
        issues_text = " ".join(semantic_check.get("issues", []))
        suggestions_text = " ".join(semantic_check.get("suggestions", []))
        return tool_response({
            "status": "rejected",
            "validation_layer": "semantic",
            "issues": semantic_check.get("issues", []),
            "tell_user": f"{issues_text} {suggestions_text}"
        })

    # Layer 4: Flag low-confidence for review
    if semantic_check["confidence"] < 0.75:
        expense = await create_expense_for_review(
            amount=amount,
            description=description,
            category=category,
            date=date,
            receipt_url=receipt_url,
            review_reason="low_validation_confidence",
            validation_details=semantic_check
        )

        return tool_response({
            "status": "pending_review",
            "expense_id": expense.id,
            "tell_user": "I've submitted your expense, but it's been flagged for manual review due to some uncertainty. You'll be notified once it's processed."
        })

    # All validation passed - check if receipt/approval needed
    rules = get_business_rules(category)

    if amount > rules.receipt_threshold and not receipt_url:
        expense_id = await create_draft_expense(
            amount=amount, description=description, category=category, date=date
        )
        return tool_response({
            "status": "needs_receipt",
            "expense_id": expense_id,
            "tell_user": f"This {category} expense of ${amount} requires a receipt. Please upload one to complete the submission."
        })

    if amount > rules.approval_threshold:
        expense = await create_expense_for_approval(
            amount=amount, description=description, category=category,
            date=date, receipt_url=receipt_url
        )
        return tool_response({
            "status": "pending_approval",
            "expense_id": expense.id,
            "approver": expense.approver_name,
            "tell_user": f"This {category} expense has been submitted for approval by {expense.approver_name}."
        })

    # Direct submission
    expense = await create_expense(
        amount=amount, description=description, category=category,
        date=date, receipt_url=receipt_url
    )
    return tool_response({
        "status": "submitted",
        "expense_id": expense.id,
        "tell_user": f"Your {category} expense of ${amount} has been submitted successfully."
    })

Look at what the agent sees: a structured response with status, an expense ID (if created), and what to tell the user. The agent never sees the expense description being validated. It never reasons about whether “asdfasdf” is meaningful. All of that happens in the tool.
The Agent’s Simplified Life

Compare the agent’s experience with and without validate-at-source:

Without Validate at Source:

User: "Log $200 expense for 'stuff'"

Agent calls: submit_expense(amount: 200, description: "stuff", category: "supplies")

Tool returns: { status: "created", expense: { id: "...", description: "stuff", ... } }

Agent reasoning:
"The expense was created, but I should check if 'stuff' is a good description...
Actually, that seems pretty vague for a $200 expense...
The finance team might reject this during audit...
I should ask the user for more details...
But wait, it's already been submitted...
Should I tell them it might be rejected later?
Or should I ask them to update it now?
What if they don't want to update it?"

Agent: "I've submitted your expense, but the description 'stuff' is quite vague. Would you like to add more detail to help with the audit process?"

User: "Nah it's fine"

Agent: [Now has to decide whether to push back or let it go...]

The agent is doing work it shouldn’t have to do. It’s examining the expense data, making judgment calls about description quality, and managing a complex conversation about something that should have been handled before submission.

With Validate at Source:

User: "Log $200 expense for 'stuff'"

Agent calls: submit_expense(amount: 200, description: "stuff", category: "supplies")

Tool returns: {
  status: "rejected",
  validation_layer: "semantic",
  tell_user: "The description 'stuff' is too vague for audit purposes. Please describe what supplies were purchased and their business purpose."
}

Agent: "The description 'stuff' is too vague for audit purposes. Please describe what supplies were purchased and their business purpose."

User: "Office supplies for the Q4 planning session"

Agent calls: submit_expense(amount: 200, description: "Office supplies for Q4 planning session", category: "supplies")

Tool returns: {
  status: "submitted",
  expense_id: "exp_123",
  tell_user: "Your supplies expense of $200 has been submitted successfully."
}

Agent: "Your supplies expense of $200 has been submitted successfully."

The agent’s job is simple: call the tool, relay the response. The validation happened in the tool. The judgment about whether “stuff” is acceptable happened in the tool. The suggestion for how to fix it came from the tool. The agent just communicates.
Semantic Validation Saves the Agent’s Attention

The key insight is that semantic validation—despite requiring LLM intelligence—doesn’t require the agent’s intelligence. A self-prompted LLM call can determine that “stuff” is too vague. It can determine that “asdfasdf” is gibberish. It can spot that $500 for “coffee meeting” is suspicious.

These judgments happen with isolated, controlled context. The self-prompted call sees only:

    The description
    The category
    The amount
    A fixed, tested prompt

It doesn’t see the conversation history. It doesn’t see what other expenses were submitted. It doesn’t carry the cognitive load of managing the user interaction. It just validates.

This keeps the agent’s attention budget focused on what the agent is actually good at: understanding the user’s intent, managing the conversation flow, and communicating naturally. The semantic judgment about whether a description is meaningful doesn’t need any of that context—it can happen in isolation.
When Semantic Validation Helps Most

Some fields need semantic validation more than others. Use it for:

Free-text descriptions. Any field where users type natural language is a candidate. “What was this expense for?”, “Please describe the issue”, “Reason for request”—these can all be validated semantically.

Category/type matching. When users select a category but also provide a description, semantic validation can check if they match. A description of “team lunch” shouldn’t be categorized as “travel.”

Reasonableness checks. “$500 for coffee” is syntactically valid but semantically suspicious. Self-prompting can flag these for review.

Policy compliance. “Client dinner at casino” might technically be valid, but policy might prohibit gambling-adjacent entertainment. Semantic validation can catch this.

Fields that rarely need semantic validation:

IDs and references. Just check if they exist.

Dates. Format validation and range checks are sufficient.

Amounts. Numeric validation and business rules cover this.

Boolean flags. Nothing to validate semantically.

Enums with fixed options. The schema handles this.

The rule: if a human reviewer would need to read and understand the field to validate it, that’s a candidate for semantic validation.
Key Takeaways

Validate at Source keeps validation work—including semantic judgment—out of the agent’s context, where it would consume attention and complicate the agent’s task.

Validation is the tool’s job. Don’t route data to the agent just so it can check if the data is valid. Validate in the tool and return a clear result.

Build a validation stack. Layer your checks from cheap to expensive: format, then business rules, then semantic. Most inputs fail early; only ambiguous cases reach the expensive layer.

Semantic validation uses self-prompting. Just because validating “asdfasdf” requires understanding doesn’t mean the agent needs to do it. An isolated, self-prompted LLM call can make that judgment.

Return actionable results. When validation fails, tell the user what’s wrong and how to fix it. The tell_user field gives the agent something it can relay directly.

Handle uncertainty gracefully. Low-confidence semantic validation doesn’t have to reject—it can flag for human review. This prevents both false positives and false negatives.

Keep the agent focused. The agent’s job is to understand the user and communicate. Validation—even complex, semantic validation—can happen invisibly in tools.
