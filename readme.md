we are going to create AI that will fill dummy data to our database and later read our database

//install package
npm init -y
npm i -D typescript ts-node @types/express @types/node
npx tsc --init
npm i langchain @langchain/langgraph @langchain/mongodb @langchain/langgraph-checkpoint-mongodb @langchain/anthropic dotenv express mongodb zod

---

1. Seeding Phase
   //seed-database.ts
   What This Code Does
   This code sets up a system that:

Creates fake employee data using AI
Stores this data in a MongoDB database
Makes it possible to search for employees based on their information using vector search

Breaking Down the Process
Step 1: Setting Up the Tools
The code starts by importing several libraries:

OpenAI tools for AI text generation and embeddings (vector representations of text)
MongoDB tools to connect to a database
Zod for validating data formats

Step 2: Defining What an Employee Record Looks Like
The code creates a detailed template (schema) that specifies exactly what information each employee record should contain:

Basic info (ID, name, birth date)
Address
Contact details
Job information
Skills
Performance reviews
Benefits
Emergency contacts

This schema acts like a contract, ensuring all employee data follows the same format.
Step 3: Generating Fake Employee Data
The generateSyntheticData() function:

Asks an AI (GPT-4o-mini) to create 10 fictional employee records
Makes sure these records follow the schema defined earlier
Returns this data as structured objects

This is useful when you're developing and testing your system before using real employee data.
Step 4: Creating Employee Summaries
For each employee, the createEmployeeSummary() function:

Takes the detailed employee record
Extracts the most important information
Combines it into a concise text summary

This summary will be used for searching later.
Step 5: Storing Data with Vector Embeddings
The seedDatabase() function:

Connects to MongoDB
Creates a database called "hr_database" with a collection called "employees"
Generates the synthetic employee data
Creates a summary for each employee
Converts each summary into a "vector embedding" (numerical representation of text)
Stores both the original data and these embeddings in the database

Why Vector Embeddings Matter
The most important concept here is vector embeddings. These are:

Mathematical representations of text that capture meaning
Allow for "semantic search" - finding similar concepts, not just exact matches
For example, if you search for "programming expert," it might find employees whose skills include "software development" even if they don't explicitly mention "programming"

Why We Need This System
This system creates a powerful HR database that:

Stores comprehensive employee information in a structured way
Allows for advanced searching beyond simple keyword matching
Can find employees based on skills, experience, or other attributes even when the exact words don't match

For example, you could ask "Who has experience managing teams in the marketing department?" and the system could find relevant employees by understanding the meaning of your query

//creating vector embeddings
How This Code Creates Embeddings

First, it creates a text summary for each employee:

The createEmployeeSummary() function combines key employee information into a single text string
This summary includes name, job title, skills, performance reviews, etc.

Then, it converts each summary into a vector embedding:

The line new OpenAIEmbeddings() creates an embeddings generator using OpenAI's embedding models
This model converts the text summary into a high-dimensional vector (typically hundreds of numbers)

Finally, it stores both the original data and the embeddings in MongoDB:

The MongoDBAtlasVectorSearch.fromDocuments() function:

Takes the record with its summary text
Generates the vector embedding
Stores everything in MongoDB
Sets up the proper indexing for vector search

---

2. Developing the AI agents
   do the mongoDB atlas search vector search setup
   the json setup:
   {
   "fields": [
   {
   "type": "vector",
   "path": "embedding",
   "numDimensions": 1536,
   "similarity": "cosine"
   }
   ]
   }

What This Code Does
This code creates an AI agent that can:

Search for employee information in a MongoDB database
Answer questions about employees using vector search
Remember conversation history even if the connection is interrupted
Make logical decisions about when to use tools vs when to just respond

Breaking Down the Process
Step 1: Setting Up the Tools & Connections
The code imports necessary libraries:

OpenAI tools for generating text responses and vector embeddings
LangGraph for creating a workflow that the AI follows
MongoDB tools for storing data and conversation history

Step 2: The callAgent Function
This is the main function that handles interactions. It takes:

A MongoDB client connection
The user's query (e.g., "Find employees with marketing skills")
A thread ID to track the conversation

Step 3: Setting Up Vector Search
The code creates a tool called employeeLookupTool that can:

Take a text query about employees
Convert it to a vector embedding (numerical representation)
Search MongoDB for similar vector embeddings
Return the most relevant employee records

javascriptCopyconst employeeLookupTool = tool(
async ({ query, n = 10 }) => {
// Initialize vector store with MongoDB
const vectorStore = new MongoDBAtlasVectorSearch(
new OpenAIEmbeddings(),
dbConfig
);

    // Search for similar employee records
    const result = await vectorStore.similaritySearchWithScore(query, n);
    return JSON.stringify(result);

},
{
name: "employee_lookup",
description: "Gathers employee details from the HR database"
// Schema defines expected parameters
}
);
Step 4: Creating a State Graph (Workflow)
The code sets up a workflow diagram (state graph) that defines how the AI agent should behave:
javascriptCopyconst workflow = new StateGraph(GraphState)
.addNode("agent", callModel) // The AI thinking node
.addNode("tools", toolNode) // The tools execution node
.addEdge("**start**", "agent") // Start with the agent
.addConditionalEdges("agent", shouldContinue) // Decide what to do next
.addEdge("tools", "agent"); // After using tools, go back to thinking
This creates a loop where the AI can:

Think about the user's question
Decide whether to use a tool or just answer
If using a tool, execute it and think again
Eventually provide a final answer

Step 5: Decision Making
The shouldContinue function controls the AI's behavior:
javascriptCopyfunction shouldContinue(state) {
const messages = state.messages;
const lastMessage = messages[messages.length - 1];

// If the AI wants to use a tool, we go to the "tools" node
if (lastMessage.tool_calls?.length) {
return "tools";
}
// Otherwise, we stop and answer the user
return "**end**";
}
Step 6: Memory & Persistence
The code uses MongoDB to remember the conversation:
javascriptCopyconst checkpointer = new MongoDBSaver({ client, dbName });
const app = workflow.compile({ checkpointer });
This means if the connection drops or the server restarts, the AI will remember the conversation history when the user returns (using the thread_id).
Step 7: Executing the Workflow
Finally, the code runs the workflow with the user's query:
javascriptCopyconst finalState = await app.invoke(
{ messages: [new HumanMessage(query)] },
{ recursionLimit: 15, configurable: { thread_id: thread_id } }
);
The recursionLimit prevents infinite loops, and the thread_id ties this to a specific conversation.
Why This Approach is Powerful

Conversational Memory: The agent remembers previous interactions in the same thread
Semantic Search: It can find relevant employees even if the query doesn't exactly match their profile
Decision Making: The agent decides when to search the database vs. when to just respond
Persistence: Conversations are saved in MongoDB and can be resumed

For example, when a user asks "Who knows about digital marketing?", the agent:

Converts this query to a vector embedding
Searches the employee database for similar vectors
Finds employees whose skills or job descriptions involve marketing concepts
Returns a helpful response with the most relevant candidates

This creates an intelligent HR chatbot that can have natural conversations about employee information while leveraging the power of vector search for more accurate results

---

3. index.ts

What This Code Does
This code sets up a web server that:

Creates a connection to MongoDB
Provides API endpoints for chatting with the AI agent
Manages conversations through unique thread IDs

Breaking Down the Process
Step 1: Setting Up the Environment
The code starts by importing necessary libraries:

dotenv/config to load environment variables
express to create a web server
MongoClient to connect to MongoDB
callAgent which is the AI agent function we saw in the previous code

javascriptCopyimport "dotenv/config";
import express, { Express, Request, Response } from "express";
import { MongoClient } from "mongodb";
import { callAgent } from "./agent";
Step 2: Creating the Express App
The code sets up an Express application to handle HTTP requests:
javascriptCopyconst app: Express = express();
app.use(express.json()); // This allows the server to parse JSON in request bodies
Step 3: MongoDB Connection
It initializes a MongoDB client using the connection string from environment variables:
javascriptCopyconst client = new MongoClient(process.env.MONGODB_ATLAS_URI as string);
Step 4: The Server Startup Function
The startServer function handles the server initialization:
javascriptCopyasync function startServer() {
try {
// Connect to MongoDB and verify the connection
await client.connect();
await client.db("admin").command({ ping: 1 });
console.log("Pinged your deployment. Successfully connected to MongoDB!");

    // Set up routes and start the server...

} catch (error) {
console.error("Error connecting to MongoDB:", error);
process.exit(1); // Exit if MongoDB connection fails
}
}
Step 5: API Routes
The code sets up three API endpoints:

Home Route - Simple health check:

javascriptCopyapp.get("/", (req: Request, res: Response) => {
res.send("LangGraph Agent Server");
});

Start Conversation - Creates a new chat thread:

javascriptCopyapp.post("/chat", async (req: Request, res: Response) => {
const initialMessage = req.body.message;
const threadId = Date.now().toString(); // Generate a unique ID based on timestamp

try {
const response = await callAgent(client, initialMessage, threadId);
res.json({ threadId, response }); // Return the thread ID so client can continue conversation
} catch (error) {
console.error("Error starting conversation:", error);
res.status(500).json({ error: "Internal server error" });
}
});

Continue Conversation - Continues an existing thread:

javascriptCopyapp.post("/chat/:threadId", async (req: Request, res: Response) => {
const { threadId } = req.params;
const { message } = req.body;

try {
const response = await callAgent(client, message, threadId);
res.json({ response });
} catch (error) {
console.error("Error in chat:", error);
res.status(500).json({ error: "Internal server error" });
}
});
Step 6: Starting the Server
Finally, the code starts the server on a specified port:
javascriptCopyconst PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
console.log(`Server running on port ${PORT}`);
});
How It All Connects

A user makes a POST request to /chat with their initial question
The server generates a unique thread ID and calls the AI agent
The agent searches the employee database and responds
The server returns both the response and the thread ID
For follow-up questions, the user makes requests to /chat/{threadId}
The agent can remember previous context because of the thread ID

This creates a stateful conversation API where:

New conversations start at /chat
Existing conversations continue at /chat/{threadId}
The MongoDB connection is shared across all conversations
Each conversation maintains its own context and history

For example, you could use it like this:

Start: POST /chat with message "Find employees with React skills"
Get back: { threadId: "1678901234567", response: "I found 3 employees..." }
Continue: POST /chat/1678901234567 with "Which one has the most experience?"
The agent remembers the previous context and can answer specifically about those 3 employees

This is essentially a backend server that makes the AI agent accessible through HTTP requests, allowing any frontend application (web, mobile, etc.) to interact with it.
