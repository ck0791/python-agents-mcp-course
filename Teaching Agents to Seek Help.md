
## Discovery Triggers: Knowing When to Ask

Training a New Employee

When you onboard a new employee, you don’t hand them a binder containing every policy, procedure, and piece of tribal knowledge the company has accumulated over the years. That would be overwhelming—and most of it wouldn’t be relevant to whatever they’re working on today.

Instead, you teach them when to ask questions.

“Before you submit an expense report, check the finance wiki.” “Before you push code, read the project’s style guide.” “When a build fails, check the troubleshooting runbook.” “If you’re not sure about a customer’s data, ask legal.”

These aren’t facts to memorize. They’re habits to develop. The new employee doesn’t need to know every expense policy—they need to know that expense policies exist and when to look them up. The knowledge comes later, at the moment it’s needed.

This is exactly how agents should work. In the previous tutorial, you learned that context is essential and that there’s too much of it to load upfront. The solution is Active Learning—discovering and loading context on demand. But Active Learning requires the agent to know when to seek context.

That’s what triggers are: the habits that tell an agent “now is the time to ask.”
What Is a Trigger?

A trigger is a condition that signals “you should look for context now.” Triggers are the agent’s instincts about when studying would help.

Think about the new employee again. After a few weeks, they develop intuitions:

    “I’m about to work in a new repository—I should check if there’s a README.”
    “This expense is unusually large—I should double-check the approval rules.”
    “That API call failed—I should look up what that error code means.”
    “I’m not sure which category this falls into—I should ask before guessing.”

These intuitions aren’t magic. Someone taught them. Either explicitly (“always check for a README”) or through experience (“I got burned last time I didn’t check”). Either way, the employee learned to recognize situations where seeking information is better than guessing.

Agents need the same teaching. Without triggers, an agent might:

    Never think to look for context (and make avoidable mistakes)
    Look for context constantly (wasting tokens and time)
    Look at the wrong times (finding irrelevant information)

Good triggers create the right habit: check for context when it matters, skip it when it doesn’t.
A Failure Without Triggers

Let’s see what happens when an agent lacks proper triggers. Imagine an agent working in the workspace from Tutorial 03:

workspace/
├── expenses/
│   ├── travel/
│   └── equipment/
├── reports/
│   └── weekly/
└── projects/
    └── alpha/

The user asks: “Add my flight to Chicago to the expenses.”

The agent has write_file and list_files tools. It doesn’t know about any context files. So it does its best:

Agent: I'll create an expense file for your Chicago flight.

[Calls write_file with path="workspace/expenses/chicago_flight.txt"
 and content="Flight to Chicago - $450"]

Done! I've added your Chicago flight expense.

The file is created. The tool worked perfectly. But:

    The filename should have been 2025-01-15_travel_chicago-flight.md
    The file is in expenses/ but should be in expenses/travel/
    The content is missing required fields: date, category, receipt status, approval status
    Expenses over $100 require a receipt URL

The agent didn’t fail because it lacked capability. It failed because it didn’t know to check for local rules before acting. It needed a trigger.
The Five Discovery Triggers

Through experience building agent systems, we’ve identified five key triggers—five situations where an agent should seek context before proceeding.
Trigger 1: Location-Based Discovery

When you enter a new directory, check for local context.

This is the most intuitive trigger. Different directories have different rules, so when you move to a new location, see if there’s documentation about how things work there.

Think of it like the new employee walking into a different department. They wouldn’t assume the same rules apply—they’d look around, check for posted guidelines, maybe ask “how do things work here?”

INSTRUCTION: When you navigate to or work within a directory, first check
if there's a .context.md or README.md file. If so, read it before taking
any actions in that directory.

The agent develops a habit: before I do anything here, let me see if there are local rules.

In code, you might encode this as:

system_prompt = """
## Working in Directories

When working in any directory:
1. First call get_directory_context(path) to check for local rules
2. If context exists, read and follow its guidelines
3. Then proceed with the user's request

Never create, modify, or delete files without first checking for
directory-specific conventions.
"""

When this trigger fires:

    Agent is asked to create a file in expenses/travel/
    Agent navigates from one project directory to another
    Agent starts working in a location it hasn’t visited before

What the agent does:

    Checks for .context.md, README.md, or similar files
    Reads and incorporates any rules it finds
    Proceeds with the task, following the local conventions

Trigger 2: Tool-Based Discovery

Before using a tool with documentation, read it.

Some tools are simple enough that the description suffices. A read_file tool does what it says. But other tools have nuances, required workflows, or edge cases that can’t fit in a brief description.

For these tools, the description itself becomes a trigger: “before you use me, read this documentation.”

@mcp.tool()
def submit_expense(expense_type: str, amount: float, description: str, receipt_url: str | None = None) -> str:
    """
    Submit an expense for reimbursement.

    IMPORTANT: Before using this tool, read the expense submission guidelines
    at resource://docs/expense-submission. This covers:
    - Required fields for each expense type
    - Valid categories and their codes
    - Receipt requirements by amount threshold
    - Approval workflows

    Submitting without understanding these guidelines will likely result in rejection.
    """
    # handler implementation
    ...

The trigger is embedded right in the tool description. When the agent reads the tool’s description (which it does during discovery), it sees the instruction to read documentation first. A well-trained agent will follow this instruction before calling the tool.

When this trigger fires:

    Agent discovers a tool with “BEFORE USING” or “IMPORTANT” documentation links
    Agent is about to use a tool it hasn’t used before in this session
    Tool description mentions prerequisites or required reading

What the agent does:

    Follows the documentation link (calls a context tool, reads a resource)
    Incorporates the guidance into its understanding
    Then proceeds to use the tool correctly

Trigger 3: Task-Based Discovery

When starting a new type of task, look for relevant procedures.

Different tasks have different requirements. Writing a report is different from filing an expense is different from reviewing code. The agent should recognize task types and seek guidance specific to that type of work.

This is like the new employee learning that “expense questions go to Finance” and “code style questions go to the tech lead.” Different domains, different sources of truth.

INSTRUCTION: When asked to perform these types of tasks, first consult
the corresponding documentation:

- Creating or modifying expenses → Read expense-procedures
- Writing reports → Read report-templates
- Modifying code in a project → Read that project's CONTRIBUTING.md
- Handling customer data → Read privacy-policies
- Deploying changes → Read deployment-checklist

This ensures you follow the correct procedures for each type of work.

The trigger creates a mapping: this kind of task needs that kind of context.

When this trigger fires:

    User asks the agent to “file an expense” (task type: expense)
    User asks the agent to “write up a weekly summary” (task type: report)
    User asks the agent to “add a feature to project alpha” (task type: code modification)

What the agent does:

    Recognizes the task type from the user’s request
    Looks up the corresponding procedures or templates
    Follows the established process for that type of work

Trigger 4: Error-Based Discovery

When something goes wrong, look for troubleshooting context.

Errors are learning opportunities. When a tool call fails, the agent shouldn’t just retry blindly or give up. It should seek guidance about what went wrong and how to fix it.

This connects directly to the Failing Forward pattern from a later tutorial. But even before we get to rich error responses, the agent can develop the habit of seeking help when things fail.

INSTRUCTION: If a tool returns an error:
1. Note the error message and any error codes
2. Call get_error_help(error_code) to find troubleshooting guidance
3. Read and follow the guidance before retrying
4. If no guidance exists, report the error to the user with full details

Don't retry blindly. Understand what went wrong first.

This turns failures into study sessions: that didn’t work—let me see if there’s documentation about why.

When this trigger fires:

    Tool returns isError: true
    Tool response contains error codes or failure messages
    An operation didn’t produce the expected result

What the agent does:

    Looks up the error code or message
    Finds troubleshooting documentation
    Follows the recovery steps
    Retries with corrected approach (or reports the issue if unrecoverable)

Trigger 5: Uncertainty-Based Discovery

When uncertain about how to proceed, seek clarification.

The agent should recognize its own uncertainty and respond by seeking information rather than guessing. This is the hardest trigger to teach because it requires self-awareness—the agent noticing that it doesn’t know something.

But it’s also one of the most valuable. An agent that guesses when uncertain will often guess wrong. An agent that asks when uncertain will usually get it right.

INSTRUCTION: If you're unsure about:
- Which category to use → Call get_valid_options("categories")
- What format is required → Call get_template(type)
- Whether an action is allowed → Call check_policy(action)
- How a field should be populated → Read the field's documentation

Don't guess. Look it up. It's better to make one extra tool call than
to make a mistake that needs to be corrected later.

When this trigger fires:

    Agent needs to choose between options but isn’t sure which is correct
    Agent is filling in a field but doesn’t know the valid values
    Agent is about to take an action but isn’t sure if it’s allowed
    Agent recognizes ambiguity in the user’s request

What the agent does:

    Identifies specifically what it’s uncertain about
    Queries for the relevant information
    Makes an informed decision (or asks the user for clarification)

Encoding Triggers in Instructions

Triggers need to be taught to the agent through its instructions. Here’s a complete example that encodes all five triggers:

## Context Discovery Protocol

You have access to tools that help you work with files and directories.
Before taking action, always ensure you have the right context.

### When Entering a Directory
Before performing any operations in a directory:
1. Call `get_directory_context(path)` to check for local rules
2. If context exists, read it carefully
3. Follow any naming conventions, required fields, or procedures specified

### When Using Tools
Some tools have detailed documentation. Check the tool description for
references like "BEFORE USING: Read..." and follow those instructions
before calling the tool.

### When Starting a Task
Different tasks have different procedures:
- Expenses → Check expense policies first
- Reports → Load the appropriate template
- Code changes → Read the project's contribution guidelines

### When Errors Occur
If an operation fails:
1. Note the error message
2. Call `get_error_guidance(error)` for troubleshooting help
3. Follow the guidance before retrying

### When Uncertain
If you're unsure about valid options, required formats, or whether
an action is allowed:
- Query for the specific information you need
- Don't guess—look it up

Remember: It's better to make one extra tool call than to make a mistake.

These instructions become part of the agent’s system prompt, shaping its behavior across all interactions.
The Workspace Example, Revisited

Let’s replay our earlier failure, but now with triggers in place.

User: “Add my flight to Chicago to the expenses.”

Agent: I need to add an expense for a Chicago flight. Let me first check
the context for the expenses directory.

[Calls get_directory_context("workspace/expenses")]

I see that expenses should be organized by type (travel, equipment, etc.)
and there's a travel subdirectory. Let me check the travel-specific rules.

[Calls get_directory_context("workspace/expenses/travel")]

I found the travel expense guidelines:
- Files must be named: YYYY-MM-DD_description.md
- Required fields: date, amount, category, description, receipt_url (if over $100)
- Category must be one of: airfare, lodging, meals, transport, other

Now I can create the expense correctly.

[Calls write_file with:
  path: "workspace/expenses/travel/2025-01-15_chicago-flight.md"
  content: properly formatted with all required fields]

Done! I've created your travel expense for the Chicago flight.
Note: Since flights are typically over $100, you'll need to add a
receipt URL before this can be approved.

The agent:

    Triggered on location — Checked expenses/ context before acting
    Triggered on task — Recognized this as a travel expense and checked travel/ context
    Applied the context — Used correct naming, location, and fields
    Proactively informed — Mentioned the receipt requirement

Same tools. Same user request. Completely different outcome.
Triggers Are Habits, Not Rules

One final note: triggers work best when they become habits, not rigid rules.

A rigid rule might say “always call get_directory_context before any file operation.” But that’s wasteful if the agent already knows the context, or if it’s doing a simple read that doesn’t need local rules.

A habit is more flexible: “when I’m about to work in a directory and I’m not sure about the local conventions, I should check.” The agent develops judgment about when checking is necessary.

This is why the instructions use phrases like “before performing operations” and “if you’re unsure” rather than “always” and “every time.” We’re training intuition, not programming a state machine.

Over time, a well-designed agent will internalize these triggers. It will check for context when it matters and skip the check when it doesn’t. That’s the goal: an agent with good research instincts.
What You’ve Learned

    Triggers are habits — They teach agents when to seek context, not what context to seek
    Five key triggers — Location, tool, task, error, and uncertainty
    Location-based — Check for context when entering a new directory
    Tool-based — Read documentation before using tools that reference it
    Task-based — Different task types need different procedures
    Error-based — When something fails, look for troubleshooting guidance
    Uncertainty-based — When unsure, look it up instead of guessing
    Instructions encode triggers — Teach the habits through the system prompt

What’s Next

The agent now knows when to ask. But where does it find answers?

In the next tutorial, Discovery Mechanisms: Knowing Where to Look, we’ll explore how agents actually find and retrieve context. Convention-based files, query tools, hierarchical context, and more. The triggers tell the agent to seek context—the mechanisms tell it where to look.

The new employee knows to ask questions. Now they need to learn who to ask and where to find the answers.
