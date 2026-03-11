
## Discovery Mechanisms: Knowing Where to Look

The Library, the Wiki, and Sarah from Compliance

In the previous tutorial, you taught your agent when to ask questions. It knows to check for context when entering a new directory, before using complex tools, when starting specific task types, after errors, and when uncertain.

But knowing when to ask is only half the equation. The agent also needs to know where to look.

Think about our new employee again. After a few days, they’ve learned the triggers: “Before submitting expenses, check the policy.” Great. But check where? Is it on the company wiki? In a shared drive? Do they need to ask Sarah from Compliance?

Different organizations have different answers:

    “Expense policies are on the wiki under Finance > Policies”
    “Each department has a README in their shared folder”
    “Ask the #finance Slack channel”
    “Check with your manager—policies vary by team”

The new employee needs to learn not just when to ask, but how the information is organized and where to find it. That’s what discovery mechanisms are: the systems and conventions that tell agents where context lives.
What Is a Mechanism?

A mechanism is how the agent actually finds and retrieves context. While triggers tell the agent when to look, mechanisms tell it where and how to look.

Mechanisms answer questions like:

    What files might contain context? What are they named?
    Is there a tool I can call to get context?
    How is context organized—flat or hierarchical?
    If context exists in multiple places, how do they relate?

A good mechanism is:

    Discoverable — The agent can find context without knowing exactly where it is
    Consistent — The same conventions work across different parts of the system
    Maintainable — Humans can update context without touching code

Let’s explore four mechanisms, starting simple and building to more sophisticated approaches.
Mechanism 1: Convention-Based Files

The simplest approach: establish a convention that certain files contain context.

In the physical world, this is like saying “every meeting room has a laminated sheet on the wall with room-specific rules.” You don’t need a central directory—just walk into any room and look for the sheet.

Common file conventions:
Filename	Purpose
.context.md	Local rules and conventions for this directory
README.md	Overview, getting started, and general guidance
.rules.json	Machine-readable constraints and validation
CONTRIBUTING.md	How to contribute to this project/space
STYLE.md	Formatting and style conventions

The agent learns: “When I need context for a directory, look for these files.”

Implementation:

CONTEXT_FILES = ['.context.md', 'README.md', '.rules.json']

def get_directory_context(directory: str) -> str | None:
    for filename in CONTEXT_FILES:
        filepath = Path(directory) / filename
        try:
            return filepath.read_text(encoding='utf-8')
        except FileNotFoundError:
            # File doesn't exist, try next
            continue
    return None  # No context file found

As an MCP tool:

@mcp.tool()
def get_directory_context(path: str) -> str:
    """
    Get context and rules for a directory.

    Looks for .context.md, README.md, or .rules.json in the specified directory
    and returns the contents. Call this before working in any directory to
    understand local conventions.

    Args:
        path: Directory path to get context for
    """
    context = get_directory_context_impl(path)

    if context:
        return f"Context for {path}:\n\n{context}"
    else:
        return f"No context file found in {path}. No special rules apply—use your best judgment."

Advantages:

    No special infrastructure needed
    Context lives alongside the content it describes
    Easy for domain experts to maintain—just edit a markdown file
    Works with existing tools (agents can use read_file if they know the convention)

Limitations:

    Agent must know the convention
    No search or fuzzy matching—you either find the file or you don’t
    Context is isolated to individual directories

Mechanism 2: Query-Based Tools

What if the agent doesn’t know exactly where context lives, but knows what it’s looking for?

Query-based tools let the agent ask for context by topic, task, or keyword. The tool handles the lookup, searching across multiple sources if needed.

This is like giving the new employee access to a company-wide search: “I need information about travel expenses” returns relevant results regardless of where those documents live.

Simple Implementation (Keyword Matching):

# A simple context registry
context_registry = {
    "expense-policies": """
## Expense Policies

All expenses require:
- Date of expense
- Amount in USD
- Category (meals, travel, supplies, software)
- Brief description
- Receipt URL for amounts over $25

Expenses over $500 require manager approval before submission.
""",
    "travel-expenses": """
## Travel Expense Guidelines

Travel expenses must be filed within 30 days of travel.
Valid categories: airfare, lodging, meals, ground-transport, other

International travel requires:
- Pre-approval from department head
- Currency conversion documentation
- Country-specific compliance acknowledgment
""",
    "report-format": """
## Weekly Report Format

Reports should include:
1. Accomplishments (what you completed)
2. In Progress (what you're working on)
3. Blockers (what's preventing progress)
4. Next Week (planned priorities)

Submit by Friday 5pm to your team's reports directory.
"""
}

@mcp.tool()
def get_context(topic: str) -> str:
    """
    Get contextual information about a topic.

    Use this when you need to understand policies, procedures, or guidelines
    for a specific type of work. Returns relevant documentation.

    Args:
        topic: What you need context about (e.g., 'expense-policies', 'travel-expenses', 'report-format')
    """
    # Direct match
    if topic in context_registry:
        return context_registry[topic]

    # Fuzzy matching - find topics that contain the search term
    matches = [
        f"### {key}\n{value}"
        for key, value in context_registry.items()
        if topic in key or key in topic
    ]

    if matches:
        return f"Found {len(matches)} related topic(s):\n\n" + "\n\n---\n\n".join(matches)

    # List available topics
    available = ", ".join(context_registry.keys())
    return f'No context found for "{topic}". Available topics: {available}'

The Power of Semantic Search

The example above uses simple keyword matching. But the most powerful implementations use vector-based semantic search. Instead of matching exact keywords, you:

    Convert all context documents into embeddings (numerical vectors that capture meaning)
    When the agent queries, convert the query into an embedding
    Find documents whose embeddings are closest to the query embedding

This means an agent asking about “reimbursement for flights” can find the “travel-expenses” document even though “reimbursement” and “flights” don’t appear in the key. The search understands meaning, not just words.

# Conceptual example - actual implementation requires an embedding service
@mcp.tool()
async def search_context(query: str) -> str:
    """Search for relevant context using natural language."""
    # Convert query to embedding vector
    query_embedding = await embeddings.encode(query)

    # Find most similar documents
    results = await vector_store.search(query_embedding, top_k=3)

    return "\n\n---\n\n".join(r.content for r in results)

We won’t cover vector search implementation in this tutorial series—it requires additional infrastructure (embedding models, vector databases). But know that this is where query-based discovery becomes truly powerful: the agent can describe what it needs in natural language, and the system finds semantically relevant context regardless of exact wording.

Advantages:

    Flexible queries—agent doesn’t need to know exact file locations
    Can search across multiple sources
    Simple version: keyword/fuzzy matching works for small context sets
    Advanced version: semantic search finds relevant content even with different wording

Limitations:

    Simple version requires maintaining a registry with good keywords
    Advanced version requires embedding infrastructure
    May return too much or too little depending on the query and threshold settings

Mechanism 3: Tree-Based Navigation

Here’s where things get powerful. Instead of flat lookups or simple inheritance, you can organize context as a tree that agents navigate—like a wiki, but with structure that enables efficient discovery.

The key insight: a tree can store massive amounts of information, but finding what you need is fast. The agent reads a node, understands what branches exist, and follows the relevant path. This is fundamentally different from searching—it’s navigation.
How Tree Navigation Works

Think about how you use Wikipedia. You don’t search for every fact. Often you:

    Start at a topic page
    Read the overview to understand the structure
    Follow links to specific sections that are relevant
    Those sections link to more detailed pages if needed

The agent does the same thing with a context tree:

knowledge/
├── index.md                    ← "Start here. Overview of all topics."
├── policies/
│   ├── index.md                ← "Company policies. See: expenses, travel, hr"
│   ├── expenses/
│   │   ├── index.md            ← "Expense rules. See: categories, approval, receipts"
│   │   ├── categories.md
│   │   ├── approval-workflow.md
│   │   └── receipt-requirements.md
│   └── travel/
│       ├── index.md            ← "Travel policies. See: domestic, international, per-diem"
│       ├── domestic.md
│       ├── international.md
│       └── per-diem-rates.md
└── procedures/
    ├── index.md
    └── ...

Each index.md serves as a navigation node. It describes what’s in that branch and tells the agent where to go next.
Two Types of Lookup

Tree navigation enables two powerful lookup patterns:

Semantic Navigation (Log-based, O(log n)): The agent reads nodes and decides which branch to follow based on understanding. It’s reading a table of contents and choosing the relevant chapter. Each step halves (roughly) the remaining search space.

Agent needs: "What are the receipt requirements for international travel?"

1. Reads knowledge/index.md → sees "policies" branch exists
2. Reads knowledge/policies/index.md → sees "expenses" and "travel" branches
3. Reads knowledge/policies/travel/index.md → sees "international.md"
4. Reads international.md → finds receipt requirements for international travel

4 reads to find specific information in a large knowledge base.

Explicit Navigation (O(1)): Index nodes can include direct pointers. “For receipt requirements, see expenses/receipt-requirements.md.” The agent doesn’t have to reason about which branch—it’s told exactly where to go.

# Travel Policies Index

## Quick Links
- Receipt requirements → ../expenses/receipt-requirements.md
- Per diem rates by country → ./per-diem-rates.md
- Pre-approval process → ../procedures/travel-approval.md

## Sections
- [Domestic Travel](./domestic.md) - Travel within the country
- [International Travel](./international.md) - Overseas travel rules

With explicit links, the agent can jump directly to what it needs—O(1) if the index points exactly where to go.
Implementation: A Navigable Context Tree

KNOWLEDGE_ROOT = Path("knowledge")

@mcp.tool()
def read_context_node(path: str) -> str:
    """
    Read a node in the knowledge tree.

    Start with 'index' to see the root. Each node may contain:
    - Content explaining that topic
    - Links to child nodes for more specific information
    - Cross-references to related topics

    Navigate by following links from node to node.

    Args:
        path: Path to the context node (e.g., 'index', 'policies/expenses/index')
    """
    node_path = path if path.endswith('.md') else f"{path}.md"
    full_path = KNOWLEDGE_ROOT / node_path

    try:
        return full_path.read_text(encoding='utf-8')
    except FileNotFoundError:
        # If exact path not found, try index.md in that directory
        index_path = KNOWLEDGE_ROOT / path / 'index.md'
        try:
            return index_path.read_text(encoding='utf-8')
        except FileNotFoundError:
            return f'No context found at "{path}". Try starting from "index" to navigate the knowledge tree.'

Example: Knowledge Tree in Action

knowledge/index.md:

# Company Knowledge Base

Welcome to the knowledge base. Navigate to find what you need.

## Main Sections

- **[Policies](./policies/index.md)** - Rules governing expenses, travel, HR, and compliance
- **[Procedures](./procedures/index.md)** - Step-by-step guides for common tasks
- **[Templates](./templates/index.md)** - Standard formats for reports, documents, and requests
- **[Reference](./reference/index.md)** - Category codes, department lists, contact information

## Quick Links

Most common lookups:
- Expense categories → [policies/expenses/categories.md](./policies/expenses/categories.md)
- Weekly report template → [templates/reports/weekly.md](./templates/reports/weekly.md)
- Approval workflows → [procedures/approvals/index.md](./procedures/approvals/index.md)

knowledge/policies/expenses/index.md:

# Expense Policies

Rules and requirements for expense submission and reimbursement.

## Sections

- **[Categories](./categories.md)** - Valid expense categories and their codes
- **[Approval Workflow](./approval-workflow.md)** - Who approves what, and thresholds
- **[Receipt Requirements](./receipt-requirements.md)** - When receipts are needed, acceptable formats
- **[Deadlines](./deadlines.md)** - Submission windows and late expense handling

## See Also

- [Travel-specific policies](../travel/index.md) - Additional rules for travel expenses
- [Expense submission procedure](../../procedures/submit-expense.md) - Step-by-step guide
- [Expense report template](../../templates/expense-report.md) - Standard format

## Key Rules

- All expenses over $25 require receipts
- Expenses over $500 require manager approval
- Expenses over $1000 require VP approval
- Submit within 30 days of the expense date

The Power of Trees

This approach scales remarkably well:
Knowledge Base Size	Flat Lookup	Tree Navigation
10 documents	Read ~5 avg	Read ~2
100 documents	Read ~50 avg	Read ~3-4
1,000 documents	Read ~500 avg	Read ~4-5
10,000 documents	Read ~5,000 avg	Read ~5-6

With a well-structured tree, the agent can navigate to specific information in a handful of reads, regardless of how large the knowledge base grows. It’s the difference between searching a library by scanning every book versus using the card catalog.
Trees Can Live Anywhere

The tree doesn’t have to be files in directories. You can implement the same structure with:

    A database: Nodes as rows with parent_id references and a content column
    A graph database: Nodes with explicit relationships
    An API: Endpoints that return node content and available links
    A wiki system: Pages with standard navigation sections

The mechanism is the structure—tree navigation with index nodes—not the storage technology.

Advantages:

    Scales to massive knowledge bases with O(log n) semantic lookup
    Explicit links enable O(1) direct jumps
    Wiki-like navigation feels natural to explore
    Can combine with inheritance (child nodes can reference parent rules)
    Storage-agnostic—works with files, databases, or APIs

Limitations:

    Requires thoughtful organization of the tree structure
    Agent must understand how to navigate (vs. just querying)
    Initial setup more complex than flat files

Mechanism 4: Resource-Based Context

MCP also supports resources—content exposed by URI that applications can list, read, and subscribe to.

Resources are fundamentally different from tools. While tools are agent-driven (the agent decides when to call them), resources are application-driven (the host application decides what to load).

Think of it like the difference between:

    Tools: The employee actively searches the wiki (agent-driven)
    Resources: The onboarding system automatically loads relevant policies into the employee’s dashboard (application-driven)

Implementation:

@mcp.resource("context://policies/expenses")
def get_expense_policy() -> str:
    """General expense submission policies and requirements."""
    return expense_policy_content


@mcp.resource("context://policies/travel")
def get_travel_policy() -> str:
    """Travel-specific expense policies and international requirements."""
    return travel_policy_content

Key characteristics:

    Resources have stable URIs (like context://policies/travel)
    Host application can list all resources and let users select which to include
    Resources can support subscriptions—notify when content changes
    Content can be dynamic (computed at read time)

When to use resources vs tools:
Use Resources When…	Use Tools When…
Host app should control what’s loaded	Agent should decide what to look up
You have a clear URI structure to expose	You need flexible, parameterized queries
Clients need to subscribe to changes	It’s a one-time lookup
Users should browse/select context	Agent should search/discover context

For most discovery patterns, tools are more natural because the agent is the one deciding it needs context. But resources work well when you want the host application to present available context for users to select.

As we discussed in Tutorial 03: don’t get hung up on the choice. What matters is that context is accessible somehow. The patterns work either way.
Combining Mechanisms

In practice, you’ll often combine mechanisms. A typical setup might include:

    Convention files for directory-specific rules (simple, co-located)
    Query tools for topic-based lookups (flexible, searchable)
    Tree navigation for large knowledge bases (scalable, structured)
    Resources for policies that the host app should surface to users

These aren’t mutually exclusive. You might use convention files for local workspace rules, a navigable tree for company-wide policies, query tools for quick lookups, and resources for context the host app pre-loads.

Example: Multi-mechanism discovery

@mcp.tool()
async def discover_context(
    directory: str | None = None,
    knowledge_path: str | None = None,
    topic: str | None = None
) -> str:
    """
    Discover relevant context for your current work.

    This tool combines multiple discovery mechanisms:
    - Convention files: checks for .context.md in the target directory
    - Tree navigation: navigates the knowledge tree if a path is given
    - Query search: finds relevant policies by topic

    Use 'directory' for local rules, 'knowledge_path' for tree navigation,
    'topic' for searches, or any combination.

    Args:
        directory: Directory to check for local .context.md
        knowledge_path: Path in knowledge tree (e.g., 'policies/expenses')
        topic: Topic to search for
    """
    results: list[str] = []

    # Mechanism 1: Convention files
    if directory:
        local_context = get_directory_context_impl(directory)
        if local_context:
            results.append(f"## Local Rules ({directory})\n\n{local_context}")

    # Mechanism 3: Tree navigation
    if knowledge_path:
        tree_content = read_context_node_impl(knowledge_path)
        if tree_content:
            results.append(f"## Knowledge: {knowledge_path}\n\n{tree_content}")

    # Mechanism 2: Query-based search
    if topic:
        search_results = search_context_impl(topic)
        if search_results:
            results.append(f'## Search Results: "{topic}"\n\n{search_results}')

    if not results:
        return "No context found. Try navigating the knowledge tree starting from 'index'."

    return "\n\n===\n\n".join(results)

The Efficiency Payoff

Why go through all this trouble? Why not just load everything into the system prompt?

Let’s do the math.

Front-loading approach:

    50 directories × average 500 tokens of context = 25,000 tokens
    Plus policy documents: another 10,000 tokens
    Total: 35,000 tokens loaded on every request
    Most of it irrelevant to any single task

Discovery approach:

    User asks to work in expenses/travel/
    Agent triggers location-based discovery
    Loads: workspace root context (~200 tokens) + expenses context (~300 tokens) + travel context (~300 tokens)
    Total: 800 tokens, all relevant

The discovery approach uses 2.3% of the tokens while providing 100% relevant context.

But it’s not just about tokens. It’s about signal vs noise. When context is loaded on demand:

    The agent sees only what matters for the current task
    There’s no confusion from irrelevant rules
    The context window isn’t cluttered with policies for systems the agent isn’t touching
    The agent can focus on the specific work at hand

Discovery is more efficient and more effective.
What You’ve Learned

    Mechanisms tell agents where to look — They complement triggers, which tell agents when to look
    Convention files are simple and co-located — .context.md next to the content it describes
    Query tools enable flexible lookups — Simple keyword matching for small sets; vector search for semantic matching at scale
    Tree navigation scales to massive knowledge bases — O(log n) semantic navigation, O(1) with explicit links
    Resources are application-driven — Good when the host app should control what’s loaded
    Combine mechanisms for comprehensive coverage — Local files + navigable trees + search + resources
    Discovery is efficient — Load only what’s relevant, not everything that exists

What’s Next

The agent now knows when to ask (triggers) and where to look (mechanisms). But what happens if it skips the research step? What if it dives straight into action without checking for context?

In the next tutorial, Just-in-Time Guidance: Guardrails for Critical Tasks, we’ll explore how the tools themselves can enforce learning. Hard reminders that block until context is read. Soft hints that guide first-time usage. Server-side guardrails that ensure agents learn before they act.

The new employee knows when to ask and where to look. But some doors require badge access—and some actions require training first.
