# Key Concepts : 

Foundry Project: Your workspace in Azure AI Foundry. Think of it as a folder that holds your agent definition, model deployment, vector stores (indexed document chunks), and conversation threads.
Agent: A configured LLM + instructions + tools (here: "File Search" tool for RAG). You define it once; the Foundry service runs it.
Thread: A single conversation session. Messages accumulate in a thread. You create one per user session.
Run: Triggered when you post a message. The Foundry service autonomously calls tools (searches your documents), generates a response, and stores all messages.
Vector Store: Where your uploaded documents are chunked, embedded, and indexed. The agent searches it automatically when answering.
Managed Identity: An Azure-native identity for your App Service. No passwords. Access to Foundry is granted via RBAC roles.

# Prerequisites to install on your machine:
- Python 3.12, Node.js 18+
- Azure CLI (az) + Azure Developer CLI (azd)
- An Azure subscription with Owner or Contributor + RBAC Administrator role
