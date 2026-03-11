
## Building a Context-Aware Workspace

The First Real Project

The new employee has completed orientation. They’ve learned:

    When to ask questions — Discovery triggers that fire before taking action
    Where to find answers — Mechanisms for locating context in files, tools, and hierarchies
    What the guardrails are — Just-in-time guidance that enforces learning for critical tasks

Now it’s time for their first real project. And like any good first project, it brings everything together.

In this tutorial, you’ll build a context-aware workspace manager—an agent that adapts its behavior based on where it’s working, loading context on demand instead of front-loading everything. You’ll see triggers, mechanisms, and guidance patterns working together in a complete system.
The Workspace

Imagine a shared workspace where different directories serve different purposes:

workspace/
├── .context.md                  ← Global rules for the entire workspace
├── expenses/
│   ├── .context.md              ← Rules for expenses
│   └── travel/
│       └── .context.md          ← Specific rules for travel expenses
├── reports/
│   └── .context.md              ← Rules for reports
└── projects/
    └── .context.md              ← Rules for project work

Each .context.md file contains rules specific to that location. Some rules inherit from parent directories, some override them, some add new constraints.

The challenge: How does an agent know which rules to follow?

The answer: It discovers them when it needs them.
The Context Discovery Server

Let’s build a server that helps agents discover context. We’ll add three key capabilities to our file server:

    get_directory_context — Read the context rules for any directory (with hierarchy)
    write_file — Create files (now we need to create, not just read)
    Enhanced list_files and read_file from before

The full code is in workspace_server.py. Here’s the key addition:

@mcp.tool()
def get_directory_context(path: str) -> str:
    """
    Get the context rules for a directory.

    This returns the .context.md file if one exists, plus any inherited
    context from parent directories.

    IMPORTANT: Always call this tool BEFORE creating or modifying files in a directory
    to understand local naming conventions, required fields, and procedures.

    Args:
        path: Directory path to get context for
    """
    contexts = get_context_hierarchy(path)

    if not contexts:
        return "No context rules found for this directory or its parents."

    return "\n\n---\n\n".join(contexts)

The magic is in get_context_hierarchy—it walks up the directory tree, collecting context files from the target directory all the way to the workspace root:

def get_context_hierarchy(target_path: str) -> list[str]:
    contexts: list[str] = []
    absolute_path = Path(target_path).resolve()
    workspace_root = Path("workspace").resolve()

    current = absolute_path

    # Walk up to workspace root, collecting context files
    while str(current).startswith(str(workspace_root)):
        context_file = current / ".context.md"
        try:
            content = context_file.read_text(encoding="utf-8")
            contexts.insert(0, f"[Context from: {current}]\n\n{content}")
        except FileNotFoundError:
            # No context file at this level, continue
            pass

        if current == workspace_root:
            break
        current = current.parent

    return contexts

This is hierarchical context discovery (Mechanism 3 from Tutorial 05)—local rules inherit from and can override parent rules.
The Context Files

Let’s create the context files that define our workspace rules. These are the “documentation” the agent will discover—written so that anyone (including agents) can understand and follow them.
Global Context (workspace/.context.md)

# Workspace Guidelines

This is a shared workspace. All files must follow these global rules:

## File Naming
- Use lowercase with hyphens (not underscores or spaces)
- Include dates in YYYY-MM-DD format where relevant
- No special characters except hyphens and periods

## Content Standards
- All text files must be UTF-8 encoded
- Use Markdown (.md) for documentation
- Include a clear title as the first line

## Process
- Check directory-specific rules before creating files
- Follow the naming conventions for each area
- Ask for clarification if requirements are unclear

Expenses Context (workspace/expenses/.context.md)

# Expense Filing Rules

Expenses must follow these rules for proper processing.

## File Naming Convention
Expense files MUST be named: `YYYY-MM-DD_vendor_category.md`

Example: `2024-01-15_delta-airlines_travel.md`

## Required Fields
Every expense file must include:
- **Date**: When the expense occurred
- **Vendor**: Who was paid
- **Amount**: In USD with two decimal places
- **Category**: Must be one of: travel, meals, equipment, software, training
- **Receipt**: Indicate if receipt is attached (yes/no)
- **Description**: Brief explanation of business purpose

## Template

Expense: [Vendor Name]

    Date: YYYY-MM-DD
    Amount: $X.XX
    Category: [category]
    Receipt: yes/no
    Description: [Business purpose]


## Approval Requirements
- Expenses over $500 require manager approval (note in description)
- Expenses over $1000 require VP approval

Travel Expenses Context (workspace/expenses/travel/.context.md)

# Travel Expense Specifics

Travel expenses have additional requirements beyond standard expenses.

## Additional Required Fields
- **Destination**: City and state/country
- **Trip Purpose**: Must reference a project or meeting
- **Per Diem Status**: Within per diem limits? (yes/no)

## Travel Categories
Use these specific categories for travel:
- `airfare` - Flights
- `lodging` - Hotels, accommodations
- `ground-transport` - Taxis, rideshare, rental cars
- `meals-travel` - Meals while traveling

## Naming Convention Override
Travel expense files should include destination:
`YYYY-MM-DD_destination_vendor_category.md`

Example: `2024-01-15_chicago_united_airfare.md`

## International Travel
If traveling internationally, also include:
- **Country**: Destination country
- **Currency**: Original currency and exchange rate if not USD
- **Visa Required**: yes/no

Reports Context (workspace/reports/.context.md)

# Report Guidelines

All reports in this directory must follow these standards.

## File Naming
Reports should be named: `YYYY-MM-DD_report-type_subject.md`

Example: `2024-01-15_weekly_project-alpha.md`

## Report Types
- `weekly` - Weekly status reports
- `monthly` - Monthly summaries
- `quarterly` - Quarterly reviews
- `incident` - Incident reports
- `analysis` - Analysis documents

## Required Sections
Every report must include:
1. **Title** - Clear, descriptive title
2. **Date** - Report date
3. **Author** - Who wrote this
4. **Summary** - Executive summary (2-3 sentences)
5. **Details** - Main content
6. **Action Items** - Any follow-up needed

## Weekly Report Template

Weekly Report: [Subject]

Date: YYYY-MM-DD Author: [Name]
Summary

[2-3 sentence overview]
Accomplishments

    [What was completed]

In Progress

    [What’s being worked on]

Blockers

    [Any impediments]

Next Week

    [Planned work]

Action Items

    [Task] - [Owner]

Projects Context (workspace/projects/.context.md)

# Project Directory Guidelines

Project files must follow development best practices.

## Structure
Each project should have:
- `README.md` - Project overview
- `docs/` - Documentation
- `src/` - Source code

## File Naming
- Code files: lowercase with hyphens
- Documentation: descriptive names with hyphens
- No version numbers in filenames (use git)

## Required Documentation
Every project must have a README.md with:
1. **Project Name** - Clear title
2. **Description** - What it does
3. **Setup** - How to get started
4. **Usage** - How to use it
5. **Contributing** - How to contribute

## Change Documentation
When making changes:
- Reference ticket numbers in commits
- Update README if behavior changes
- Add/update tests for new functionality

The Context-Aware Agent

Now let’s build an agent that uses these context files. The key difference from our basic agent is the discovery trigger—before acting in a directory, the agent checks for context.

The full code is in workspace_agent.py. Here’s the system prompt that encodes the discovery pattern:

system_prompt = """You are a workspace assistant that helps manage files and content.

## CRITICAL: Context Discovery Protocol

Before creating or modifying ANY file, you MUST:
1. Call get_directory_context(path) for the target directory
2. Read and understand ALL context rules (global AND local)
3. Follow the naming conventions and required fields exactly
4. Use the templates provided in the context

This is not optional. The context files contain rules that MUST be followed.

## When Asked to Create Files

1. First, ask where the file should go if not specified
2. Call get_directory_context for that location
3. Follow the naming convention from the context
4. Include all required fields from the context
5. Use the template if one is provided

## When Asked About Rules

If the user asks about rules or conventions:
1. Call get_directory_context for the relevant directory
2. Summarize the rules clearly
3. Point out any inheritance from parent directories

## Available Tools
- list_files: See what's in a directory
- read_file: Read file contents
- write_file: Create or update files
- get_directory_context: Get rules for a directory (USE THIS FIRST!)

Remember: Context discovery is not optional. Always check before acting."""

The prompt instills the habit from Tutorial 04: “Before I do anything here, let me see if there are rules.”
Seeing It in Action

Let’s trace through a real interaction. The user asks:

    “Create an expense for my flight to Chicago on Delta”

Without Context Discovery:

Agent: [creates file "chicago_flight.md"]
       [writes "Delta flight to Chicago - $450"]

Result: Wrong filename, wrong format, missing required fields

With Context Discovery:

Agent: I need to create an expense file. Let me check the rules first.

[calls get_directory_context("workspace/expenses/travel")]

Agent: I see. Travel expenses need:
       - Filename: YYYY-MM-DD_destination_vendor_category.md
       - Required fields: Date, Vendor, Amount, Category, Receipt,
         Description, Destination, Trip Purpose, Per Diem Status

Agent: I'll need some more information. What was the date and amount?
       What project or meeting was this for?

User: January 15th 2024, $450, for the Q1 planning meeting

[calls write_file with proper format]

Agent: Created: workspace/expenses/travel/2024-01-15_chicago_delta_airfare.md

Contents:
# Expense: Delta Airlines

- **Date**: 2024-01-15
- **Amount**: $450.00
- **Category**: airfare
- **Receipt**: no
- **Destination**: Chicago, IL
- **Trip Purpose**: Q1 Planning Meeting
- **Per Diem Status**: yes
- **Description**: Flight to Chicago for Q1 planning meeting with leadership team

The agent discovered the rules, followed them, and asked for missing information. That’s context-aware behavior.
Running the Workspace Agent

To try this yourself:

cd python-code
pip install -r requirements.txt
python workspace_agent.py "Create an expense for my $200 dinner at Fancy Restaurant for client entertainment"

Watch how the agent:

    Identifies this is an expense
    Calls get_directory_context for the expenses directory
    Reads the naming conventions and required fields
    Creates a properly formatted file

Try these other prompts:

python workspace_agent.py "What are the rules for travel expenses?"
python workspace_agent.py "Create a weekly report for Project Alpha"
python workspace_agent.py "Help me file an expense for my Uber from the airport"

The Power of Hierarchical Context

Notice how the travel expense rules extend the general expense rules. When the agent calls get_directory_context("workspace/expenses/travel"), it receives:

    Global workspace rules (from workspace/.context.md)
    General expense rules (from workspace/expenses/.context.md)
    Travel-specific rules (from workspace/expenses/travel/.context.md)

The agent sees the full inheritance chain, understanding that:

    Global naming conventions (lowercase with hyphens) still apply
    General expense requirements (amount, vendor, category) are inherited
    Travel adds extra requirements (destination, trip purpose)
    Travel overrides the naming convention (adding destination)

This is contextual inheritance—rules flow down and can be extended or overridden at each level.
Key Implementation Patterns
Pattern 1: Discovery Before Action (Trigger)

The system prompt explicitly requires context discovery before any file operation:

Before creating or modifying ANY file, you MUST:
1. Call get_directory_context(path) for the target directory

This is the location-based trigger from Tutorial 04. The agent doesn’t “sometimes” check for context—it always does.
Pattern 2: Hierarchical Discovery (Mechanism)

The get_directory_context tool does more than read a single file—it walks the hierarchy:

# Walk up to workspace root, collecting context files
while str(current).startswith(str(workspace_root)):
    context_file = current / ".context.md"
    # ... collect context ...
    current = current.parent

This is the hierarchical mechanism from Tutorial 05. The agent gets complete context with a single tool call.
Pattern 3: Instructive Context Files

The .context.md files are written to be instructive, not just documentary:

    They use templates the agent can copy
    They explain why rules exist
    They provide examples of correct naming
    They specify required vs. optional fields

Good context files teach the agent what to do.
Pattern 4: Prevention Through Discovery

By checking rules before acting, the agent prevents errors rather than recovering from them. Compare:

Without discovery: Act → Fail → Try to understand error → Retry With discovery: Learn → Act correctly the first time

Prevention is more efficient than recovery (though Failing Forward helps when prevention isn’t enough).
The Payoff: Separation of Concerns

Look at what we’ve achieved. The write_file tool knows nothing about expense policies, report formats, or project conventions. It just writes files. All the domain-specific knowledge lives in .context.md documents that:

    Anyone can edit — No coding required, just markdown
    Change independently — Update expense policies without touching code
    Domain experts can own — Finance maintains expense rules, not developers

This is the payoff of Active Learning that we discussed in Tutorial 03: late binding lets you change how tools are used without changing the tools themselves. The finance team updates expenses/travel/.context.md, and every agent immediately follows the new rules. No deployments. No code reviews. No risk of breaking something else.

Tools are the vocabulary. Context files are the style guide. Keep them separate, and both become easier to maintain.
Extending the Pattern

This workspace example demonstrates the fundamentals. Here’s how you might extend it:
Multiple Context Sources

Instead of just .context.md files, check multiple sources:

    .context.md for human-readable rules
    .rules.json for machine-parseable constraints
    README.md as a fallback

Dynamic Context

Context doesn’t have to be static files:

@mcp.tool()
async def get_project_context(project_id: str) -> str:
    """Fetch project-specific context from database."""
    rules = await db.get_project_rules(project_id)
    return rules

Context Validation

Add a tool that validates content against context rules before writing:

@mcp.tool()
async def validate_file(path: str, content: str) -> str:
    """Validate file content against context rules."""
    rules = await get_context_rules(path)
    violations = check_violations(content, rules)
    return format_validation_results(violations)

Just-in-Time Enforcement

Add hard reminders (from Tutorial 06) for critical operations:

@mcp.tool()
def delete_file(path: str) -> str:
    """Delete a file (requires acknowledgment of deletion policy)."""
    if not has_acknowledged(SESSION_ID, "deletion-policy"):
        return "⚠️ Read deletion-policy first."
    # Proceed with deletion
    ...

User Preference Context

Layer user preferences on top of workspace rules:

contexts = get_context_hierarchy(dir_path)
user_prefs = await get_user_preferences(user_id)
# Merge user preferences with directory context

What You’ve Built

A complete context-aware workspace system with:

    Hierarchical context files that define rules at each directory level
    A discovery tool that retrieves context with inheritance
    A context-aware agent that checks rules before acting
    Proper file creation that follows discovered conventions

This is Active Learning in action: the agent discovers what it needs to know exactly when it needs to know it.
What You’ve Learned

    Context lives alongside content — .context.md files in each directory
    Hierarchical inheritance — Rules flow from parent to child directories
    Discovery triggers — System prompts can instill “check first” habits
    Context-aware tools — Tools that aggregate context from multiple levels
    Prevention beats recovery — Learn the rules before acting, not after failing

