
The Agent Loop: How AI Agents Work
Connecting to the Conversation

In the last tutorial, you built a Tool Server—the computer’s side of the conversation. You created a program that exposes capabilities and waits for requests. But a conversation needs two parties.

Now you’ll build the other side: the agent.

By the end of this tutorial, you’ll have a working AI agent that can explore your filesystem by asking questions and using tools. More importantly, you’ll understand the fundamental pattern that every AI agent follows—a pattern so universal that once you see it, you’ll recognize it everywhere.
The Universal Pattern

Every AI agent, no matter how sophisticated, follows the same basic loop:

PERCEIVE → DECIDE → ACT → OBSERVE → REPEAT

This is the Agent Loop. It’s the architecture of intelligence in action.

Think about how you accomplish tasks yourself. You don’t jump into action blindly. You:

    Perceive — Look around, gather information, understand the situation
    Decide — Based on what you know, choose what to do next
    Act — Execute your chosen action
    Observe — See what happened as a result
    Repeat — Loop back with new information

AI agents work exactly the same way. The difference is:

    Instead of eyes, they have tool discovery
    Instead of hands, they have tool execution
    Instead of intuition, they have language models

But the loop is identical.
The Loop Visualized

                    ┌─────────────┐
                    │   START     │
                    └──────┬──────┘
                           │
                           ▼
              ┌────────────────────────┐
              │       PERCEIVE         │
              │  • Discover tools      │
              │  • Gather context      │
              └───────────┬────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │        DECIDE          │
              │  • LLM reasons         │
              │  • Selects tool        │
              │  • Chooses parameters  │
              └───────────┬────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │         ACT            │
              │  • Execute tool        │
              │  • Await response      │
              └───────────┬────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │       OBSERVE          │
              │  • Process result      │
              │  • Update context      │
              │  • Check completion    │
              └───────────┬────────────┘
                          │
                          ▼
                    ┌──────────┐
                    │  Done?   │
                    └────┬─────┘
                         │
              ┌──────────┴──────────┐
              │ NO                  │ YES
              ▼                     ▼
        [Loop back to         ┌──────────┐
         DECIDE]              │  FINISH  │
                              └──────────┘

Let’s build each phase, step by step. You’ll find the complete working code in the python-code/ folder—we’ll walk through the key parts here.
Phase 1: PERCEIVE — Discovering What’s Possible

Before your agent can do anything, it needs to know what it can do. This is the perception phase—the agent opens its eyes and surveys the landscape of available capabilities.

In MCP, this means connecting to a Tool Server and asking: “What tools do you have?” Think of it as the agent introducing itself and learning what topics this computer is willing to discuss.

# Connect to the server
server_params = StdioServerParameters(
    command="python",
    args=["server.py"],
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as client:
        await client.initialize()

        # PERCEIVE: Discover what tools exist
        tools_result = await client.list_tools()
        tools = tools_result.tools

When you run the agent, you’ll see:

Connected to server!

Available tools:
  - list_files: List files and directories in a given path
  - read_file: Read the contents of a text file

This is the agent opening its eyes. The list_tools() call returns every tool the server exposes, along with descriptions and parameter schemas. The agent now knows what this computer can do—and therefore, what kinds of requests it can make in their conversation.
Phase 2: DECIDE — Choosing What to Do

Now the agent knows what tools exist. But how does it decide which one to use? And with what parameters?

This is where the language model comes in. The LLM is the decision engine—it reads the available tools, understands the user’s goal, and figures out what to say to the computer. It’s composing the next message in the conversation.

The code in llm.py provides a simple abstraction that works with OpenAI, Anthropic, or Google. The key class is straightforward:

class LLMProvider(ABC):
    @abstractmethod
    async def chat(self, messages: list[Message], tools: list[Tool] | None = None) -> LLMResponse:
        pass

You give it the conversation history and available tools. It returns either text content or tool calls—the LLM’s decision about what to do next.

# DECIDE: Ask the LLM what to do
response = await llm.chat(messages, tools)

if response.tool_calls:
    # LLM decided to use a tool
    print(f"LLM decided to use: {response.tool_calls[0].name}")
else:
    # LLM decided no tool is needed—it has an answer
    print(response.content)

The LLM reads the tool descriptions and schemas, considers the user’s message, and decides which tool (if any) to call. You didn’t hardcode this logic—the LLM figured it out.
Phase 3: ACT — Executing the Decision

Once the LLM decides what to do, the agent executes it. This is the ACT phase—taking the decision and making it real.

# ACT: Execute the tool the LLM chose
result = await client.call_tool(
    tool_call.name,
    tool_call.arguments,
)

This is the agent speaking to the computer—sending its request and waiting for a response. The call_tool method delivers the message to the MCP server, which does the actual work (reading a file, listing a directory, whatever the tool does) and sends back a reply.

Notice the separation: the LLM decides, but your code executes. The LLM doesn’t call client.call_tool() directly. You do. This separation is powerful—it means you can add logging, validation, rate limiting, or any other control logic between decision and action.
Phase 4: OBSERVE — Processing the Result

After acting, the agent observes what happened. This is the agent listening—hearing back from the computer what it did, what it found, or what went wrong.

# OBSERVE: Process result and add to conversation
result_text = get_result_text(result)

# Add to conversation history for next iteration
messages.append(Message(
    role="assistant",
    content=f"I'll use the {tool_call.name} tool.",
))
messages.append(Message(
    role="user",
    content=f"Tool result:\n{result_text}",
))

This is the critical insight: the computer’s response becomes part of the ongoing conversation. The agent doesn’t just fire-and-forget—it listens to what the computer said, incorporates that into its understanding, and uses it to decide what to say next. Just like any good conversation, each exchange builds on the last.
The Complete Loop

Now let’s see all four phases working together. The full code is in agent.py, but here’s the core loop:

# THE AGENT LOOP
while iteration < max_iterations:
    iteration += 1

    # DECIDE: Ask LLM what to do
    response = await llm.chat(messages, tools)

    if response.tool_calls:
        # ACT: Execute each tool call
        for tool_call in response.tool_calls:
            result = await client.call_tool(
                tool_call.name,
                tool_call.arguments,
            )

            # OBSERVE: Process result and add to conversation
            result_text = get_result_text(result)
            messages.append(Message(role="assistant", content=f"I'll use {tool_call.name}."))
            messages.append(Message(role="user", content=f"Tool result:\n{result_text}"))
        # REPEAT: Loop continues
    else:
        # No tool call means the LLM is done
        print(response.content)
        break

That’s the entire pattern. PERCEIVE happens once at startup (discovering tools). Then the loop runs: DECIDE → ACT → OBSERVE → REPEAT until the LLM decides it has enough information to answer.

Each iteration is a turn in the conversation with the computer. The agent speaks (ACT), listens (OBSERVE), thinks about what it heard (DECIDE), and speaks again. The conversation continues until the agent has learned enough to help the user.
Running Your Agent

From the python-code/ folder, set your API key and run:

cd python-code
pip install -r requirements.txt

export ANTHROPIC_API_KEY=sk-ant-...
# or
export OPENAI_API_KEY=sk-...

python agent.py "What files are here and what's in requirements.txt?"

Watch what happens:

Using Anthropic Claude
Connecting to MCP server...

Discovered 2 tools: list_files, read_file

--- Iteration 1 ---
Calling: list_files({"path":"."})
Result preview: 📁 __pycache__
📁 venv
📁 workspace
📄 agent.py
📄 llm.py
📄 requirements.txt
📄 server.py

--- Iteration 2 ---
Calling: read_file({"path":"requirements.txt"})
Result preview: mcp
httpx
python-dotenv
pydantic

--- Iteration 3 ---

=== Final Response ===
Here's what I found:

**Files in current directory:**
- __pycache__/ (directory)
- venv/ (directory)
- workspace/ (directory)
- agent.py
- llm.py
- requirements.txt
- server.py

**requirements.txt contents:**
This is a Python project with the following dependencies:
- mcp: The Model Context Protocol SDK
- httpx: Async HTTP client for API calls
- python-dotenv: Environment variable loading
- pydantic: Data validation

The agent:

    Perceived — Discovered list_files and read_file
    Decided — Chose to call list_files first
    Acted — Executed the tool
    Observed — Processed the result
    Repeated — Decided to read requirements.txt
    Finished — Synthesized a final answer

You didn’t tell it to call list_files first. The LLM figured that out. That’s what makes it an agent.
The Key Insight: The LLM is Not the Agent

Many developers make the mistake of thinking the LLM is the agent. It’s not.

The LLM is the decision engine within the agent. The agent is the entire loop—the orchestration of perception, decision, action, and observation.

Look at the code again:

while iteration < max_iterations:
    # DECIDE - the LLM does this
    response = await llm.chat(messages, tools)

    if response.tool_calls:
        # ACT - your code does this
        result = await client.call_tool(...)

        # OBSERVE - your code does this
        messages.append(...)

The LLM doesn’t call client.call_tool(). You do. The LLM just tells you what it thinks should happen. This separation is powerful because:

    You control the loop — You decide how many iterations, what termination conditions, what safety checks
    Memory lives outside the LLM — The messages list persists across iterations
    Tools are separate from reasoning — The LLM decides, but your code executes

Try Different Questions

python agent.py "Find all Python files and tell me what they do"
python agent.py "What's in the workspace directory?"
python agent.py "Read llm.py and explain what it does"

Watch how the agent chooses different tools and parameters based on what you ask. It’s reasoning about each question and deciding what actions to take.
Building a Research Agent

The agent.py in the code folder is actually configured as a research agent—it can explore a folder and answer questions by reading multiple files and synthesizing information.

Try asking it research questions:

python agent.py "How is this project structured? What does each file do?"

The agent will explore the folder, read multiple files, and synthesize an answer. It’s doing research—gathering information from multiple sources and synthesizing it. The only difference from a basic agent is the system prompt that tells it how to approach research tasks.
What You’ve Built

A complete AI agent that:

    Perceives available tools through MCP
    Decides what to do using an LLM
    Acts by executing tools
    Observes results and continues until done
    Researches questions using files in a folder

This is a real, working agent. The same pattern works for any tools you create—database queries, API calls, file operations, web searches. The loop is universal.
What You’ve Learned

    Every agent is a loop — PERCEIVE → DECIDE → ACT → OBSERVE → REPEAT
    PERCEIVE means discovering tools with list_tools()
    DECIDE means asking the LLM what to do
    ACT means executing tools with call_tool()
    OBSERVE means adding results to context for the next decision
    The LLM is not the agent — It’s the decision engine within the loop you control

What’s Next

You now have the fundamentals. You can build agents that perceive, decide, act, and observe. But there’s more to learn:

    Response-as-Instruction — How tool responses can guide agent behavior beyond just returning data
    Failing Forward — Using errors as teaching moments that help agents recover and succeed

These patterns build on the foundation you’ve established. The Agent Loop is the chassis. These patterns are the upgrades that make your agents smarter and more capable.

But first: experiment with what you have. Add new tools. Try different prompts. See what the agent can do—and where it gets confused. That experience will make the advanced patterns click.

The conversation with the computer has begun. Now it’s time to make it smarter.
