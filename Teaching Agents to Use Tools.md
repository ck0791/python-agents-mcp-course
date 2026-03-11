
## Context is Everything

The Problem with Tool Descriptions

In the previous tutorials, you built an agent that can explore files and answer questions. It works—but there’s a fundamental limitation hiding in plain sight.

When your agent discovered the read_file and list_files tools, it learned what they do. The descriptions said “list directory contents” and “read file contents.” That’s enough for simple exploration.

But what happens when how to use a tool depends on where you’re using it? Or what you’re trying to accomplish? Or who you’re working with?

Tool descriptions can tell you what a tool does. They can’t tell you when to use it, why to use it in a particular way, or how to interpret what comes back.

That requires context.
A Tale of Two Directories

Imagine you’re building an agent that helps manage a workspace with different folders for different purposes:

workspace/
├── expenses/
│   ├── travel/
│   ├── equipment/
│   └── meals/
├── reports/
│   ├── weekly/
│   └── quarterly/
└── projects/
    ├── alpha/
    └── beta/

Your agent has the same tools everywhere: list_files, read_file, write_file. But the rules are completely different:

In expenses/travel/:

    Files must be named YYYY-MM-DD_destination_purpose.md
    Each file must contain: date, amount, category, receipt status, approval status
    Expenses over $500 require manager approval documentation
    International travel requires additional compliance fields

In reports/weekly/:

    Files must be named YYYY-WW_report.md (year and week number)
    Reports must include: accomplishments, blockers, next week’s priorities
    Must be submitted by Friday 5pm
    Must mention any at-risk deliverables

In projects/alpha/:

    All code changes must reference a ticket number
    Documentation must follow the team’s style guide
    Test files must accompany any new functionality
    The project has specific security review requirements

The tools are identical. The context is completely different.
What the Agent Doesn’t Know

Without context, the agent has to guess:

User: "Add my flight to Chicago to the expenses"

Agent: [creates file "chicago_flight.txt"]
       [writes "Flight to Chicago - $450"]

System: ❌ Wrong filename format
        ❌ Missing required fields
        ❌ Wrong file extension
        ❌ Missing receipt status

The agent used the tools correctly! write_file worked exactly as documented. But the agent didn’t know the rules of this particular location.

This isn’t a tool problem. It’s a context problem.
Types of Context

Here’s the key insight: context isn’t just one thing. When we talk about “context,” we’re actually talking about several distinct types of information, each serving a different purpose.

Think about what you’d need to know as a new employee trying to file an expense report. You’d need to know where to put the file and what to name it (structural). You’d need to understand the steps to get it approved (procedural). You’d need to know the limits—what’s allowed without extra approval (policy). You’d need to understand the vocabulary—what “category codes” mean and which ones to use (domain). You might need to know your manager’s preferences for how they like things formatted (preference). And it would help to know why certain rules exist—the history behind them (historical).

Each type answers a different question:
Type	What It Answers	Example
Structural	Where does this go? What’s it called?	”Expense files use YYYY-MM-DD_ prefix”
Procedural	How do I do this? What’s the process?	”Submit → Review → Approve → Reimburse”
Policy	What are the rules? What’s allowed?	”Max $75/day for meals without receipt”
Domain	What does this mean? What are the options?	”Category codes: TRV, EQP, MLT, SFT”
Preference	How do they like it? What’s the style?	”Use bullet points, not paragraphs”
Historical	What happened before? Why is it this way?	”We require photos since the scanner broke”

Different tasks require different combinations. An agent creating an expense needs structural context (where to put it, what to name it), policy context (what’s allowed), and domain context (valid categories). An agent writing a report needs procedural context (what sections to include) and preference context (how the reader likes things formatted). Recognizing these types helps you think about what context your agent will need for the tasks it performs.
The Scale Problem

Here’s the challenge: there’s too much context to load upfront.

Think about a real organization. There might be dozens of project directories, each with its own conventions and requirements. Hundreds of policy documents covering everything from expense limits to data handling to communication guidelines. Thousands of historical decisions that explain why things are done a certain way. Countless individual preferences about formatting, style, and workflow.

Now imagine trying to cram all of that into a system prompt. Even if you could fit it—and you probably can’t, given context window limits—most of it would be irrelevant to any particular task. If the user wants to file a travel expense, they don’t need to know the code review guidelines for Project Alpha. If they’re writing a weekly report, they don’t need the international travel compliance requirements.

Loading everything upfront wastes tokens on irrelevant information, clutters the context window with noise, and makes it harder for the agent to find the signal. It’s like giving someone the entire employee handbook when they just need to know how to book a conference room.

What you need is contextual loading—the ability to discover and load only the context that’s relevant to what you’re doing right now. Load the travel expense rules when you’re in the travel expenses folder. Load the report format when you’re writing a report. Keep everything else out of the way until it’s needed.
The Active Learning Pattern

This brings us to Active Learning: instead of front-loading all possible context, give the agent the ability to discover what it needs, when it needs it.

The pattern has two parts: discovery mechanisms (how the agent finds context) and discovery triggers (when the agent looks for it).
1. Discovery Mechanisms

Discovery mechanisms are the ways an agent can actually obtain context once it decides to look. The simplest approach is to look for context files in the current directory—a .context.md file sitting alongside the content, explaining the local rules. This works well for location-based context and is easy for humans to maintain.

But context doesn’t always live in files. You might query a context service that returns policies based on the current user, project, or time of year. You might read documentation linked from tool descriptions—the tool says “see the style guide at…” and the agent follows the link. Or you might follow references from one piece of context to another, building up a complete picture by traversing a web of related documents.

The mechanism you choose depends on where your context naturally lives. If domain experts maintain markdown files in directories, file-based discovery makes sense. If policies live in a database, a query service might be better. The point is to have some mechanism the agent can use to pull in context on demand.
2. Discovery Triggers

Triggers tell the agent when to look for context. Without triggers, the agent might never think to check—or might check constantly, wasting time and tokens. Good triggers create a habit: “When X happens, look for context.”

Some triggers are location-based: entering a new directory should prompt the agent to check for local rules before doing anything. Some are tool-based: before using a tool for the first time, check if there’s documentation about how to use it well. Some are error-based: when something fails, look for troubleshooting guides that might explain what went wrong. And some are task-based: when starting a new type of work (writing a report, filing an expense), look for the procedures that govern that type of work.

The triggers you define shape your agent’s behavior. An agent with a “check before creating files” trigger will discover naming conventions before making mistakes. An agent without that trigger will guess—and often guess wrong.
Why This Works

The key insight: the rules describing how and when to discover context are more token-efficient than loading all context upfront.

If you need to work in expenses/travel/, you load the travel expense context. If you need to work in projects/alpha/, you load the project alpha context. You never load both unless you need both.
Why Context Lives in Documents, Not Tool Descriptions

You might be tempted to solve this problem by putting all the context into tool descriptions. Just make the write_file tool description say “when writing expense files, use this naming convention, include these required fields, follow this approval process…”

Don’t do this. Here’s why.
Tool Descriptions Should Be Stable

Tool descriptions need to focus on what the tool does—its fundamental, unchanging behavior. A write_file tool writes files. That’s what it does, regardless of whether you’re writing expense reports, weekly summaries, or project documentation.

The moment you embed process details into tool descriptions, you’ve mixed things that change at different rates. The tool’s core behavior rarely changes. Expense policies change quarterly. Approval thresholds change when budgets shift. Required fields change when compliance requirements update. Baking frequently-changing information into tool descriptions creates a maintenance nightmare—every policy change means updating code, redeploying servers, and hoping you didn’t break something.
Late Binding Is Powerful

Active Learning gives you late binding—the ability to attach different contextual knowledge to the same tool based on the situation. The write_file tool doesn’t know about expense policies. But when the agent is working in the expenses directory, it discovers the expense context and applies it.

This means you can change how a tool gets used without changing the tool itself. Update a context document, and every agent immediately follows the new rules. No code changes. No deployments. No risk of breaking the tool for other use cases.
Documents Are Easier to Maintain

There’s a practical reality at play: documents are much easier to maintain than code.

When context lives in a markdown file, anyone with a text editor can update it. When that same context is embedded in a tool description inside Python code, you need a developer. The feedback loop is slower. The barrier to making corrections is higher. Small errors persist longer because fixing them feels like a bigger undertaking than it should be.
Documents Enable Collaboration

Here’s the deeper benefit: documents are human-readable.

When context lives in plain language documents, domain experts can participate directly. The finance team can write and maintain expense policies. The legal team can own compliance requirements. Project managers can define their project’s conventions. Nobody needs to learn Python or understand JSON schemas.

This is transformative. Instead of funneling all knowledge through developers who translate requirements into code, you let the people who actually understand the domain maintain the context directly. The agent reads what the experts write.

Active Learning isn’t just an architectural pattern—it’s a way to democratize the management of agent behavior.

    Note: In our examples, you’ll see context embedded in tool descriptions—patterns like “Domain experts can write and maintain expense policies.” In the real world, you would keep these in separate documents that domain experts can edit directly, then load them dynamically.

How to Expose Context: Resources vs Tools

The goal is simple: give the agent a way to pull in context when it needs it. MCP offers two mechanisms for exposing information, and understanding them helps you choose the right approach for your situation.
MCP Resources

Resources expose content by URI—things like files, database records, or policy documents:

@mcp.resource("context://policies/travel")
def get_travel_policy() -> str:
    """Travel policy documentation."""
    return travel_policy_content
# Client reads: await client.read_resource("context://policies/travel")

Despite the name, resources aren’t necessarily static. A resource can return dynamically computed content—the current state of a system, live data from a database, or policies that vary based on the current user. Resources can also support subscriptions, notifying clients when content changes so they can reload it. And they can be listed and filtered by the host application, letting users browse what’s available.

The key characteristic: resources are application-driven. The host application (Claude Desktop, an IDE, your custom interface) typically presents available resources to the user or decides which resources to include in context. The agent doesn’t “call” resources the way it calls tools.
MCP Tools

Tools are functions the agent calls directly:

@mcp.tool()
def get_directory_context(path: str) -> str:
    """Get context for a directory."""
    ...
# Agent calls: await client.call_tool("get_directory_context", {"path": "..."})

The key characteristic: tools are agent-driven. The agent decides when to call them, with what parameters, as part of its reasoning loop.
The Practical Reality

Here’s the thing: everything you can do with a resource, you can do with a tool that returns the same information.

Want to expose travel policies? You could:

    Create a resource at context://policies/travel
    Create a tool called get_policy that takes a topic parameter

Both work. The agent gets the same context either way.

In practice, many implementations just use tools for everything. The mental model is simpler—agents call tools, that’s all there is to it. Tools are also more flexible, since they can take parameters to customize what gets returned. And using tools keeps everything consistent: the agent’s entire vocabulary of actions is just “call tools.”

That said, resources shine in certain situations. If you have a clear URI structure you want to expose—like a hierarchy of policy documents—resources make that structure explicit and browsable. If you need clients to be notified when content changes, resources support subscriptions that tools don’t naturally have. And if you want the host application (not the agent) to control what context gets loaded—perhaps letting users select which project’s rules to apply—resources give that control to the application.

For context discovery patterns specifically, tools often make more sense. The agent is the one deciding it needs context, so it makes sense for the agent to call a tool to get it. But the choice isn’t critical—the patterns work either way.
The Bottom Line

Don’t get hung up on resources vs tools. What matters is that you expose context somehow, give the agent a way to discover it, and load it on demand rather than upfront. Whether that context comes from a resource read or a tool call is an implementation detail.

The patterns in this tutorial work with either approach. We’ll use tools in our examples because they’re simpler and more universal, but the concepts apply regardless of which mechanism you choose.
What You’ve Learned

    Tool descriptions aren’t enough — They tell what, not when, why, or how
    Context varies by location — The same tool needs different usage patterns in different places
    Context has types — Structural, procedural, policy, domain, preference, historical
    You can’t load everything — There’s too much context for any single prompt
    Active Learning is the solution — Discover and load context on demand

