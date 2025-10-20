# AzureMLProjectDIO

Here is a detailed, step-by-step project plan for building your interactive PDF chat system using the Azure Machine Learning (Azure ML) ecosystem.

This plan focuses on using the Azure ML Workspace as the central hub for development, orchestration, and deployment, integrating with other necessary Azure services.

-----

### **Project Roadmap: Azure ML-Powered RAG System for TCC Research**

**Objective:** To build and deploy a "Chat with your PDFs" application entirely within the Azure ecosystem, using Azure Machine Learning as the core platform for development, data processing, model orchestration (RAG), and endpoint deployment.

**Core Azure Services:**

  * **Azure Machine Learning Workspace:** The central hub for all activities.
  * **Azure Blob Storage:** To store the source PDF documents.
  * **Azure OpenAI Service:** To access embedding models and Generative LLMs (e.g., GPT-4, GPT-3.5-Turbo).
  * **Azure AI Search (formerly Cognitive Search):** To serve as the persistent, scalable Vector Store.
  * **Azure App Service:** To host the final interactive chat web application.

-----

### **Phase 0: Foundation & Prerequisite Setup**

This phase is about preparing your Azure environment.

1.  **Create Core Resources:**
      * Create a new **Resource Group** (e.g., `TCC-Chat-RG`) to hold all project assets.
      * Request **Azure OpenAI Service access** for your subscription. This is a crucial, manual approval step. You will need quota for both an *embedding model* and a *chat model*.
2.  **Provision Azure ML Workspace:**
      * Create your **Azure Machine Learning Workspace** (e.g., `tcc-ml-workspace`). This will automatically provision associated resources:
          * Azure Storage Account (for notebooks, data, and models)
          * Azure Key Vault (for secrets)
          * Azure Application Insights (for monitoring)
3.  **Set Up AI Services:**
      * Create your **Azure OpenAI Service** resource (e.g., `tcc-openai-service`).
      * Create your **Azure AI Search** resource (e.g., `tcc-ai-search`). Select a tier that supports vector search (e.g., Basic or Standard).
4.  **Configure Development Environment:**
      * Open your Azure ML Studio.
      * Create a new **Compute Instance** (e.g., a `Standard_DS3_v2`) to serve as your personal development machine for running notebooks and scripts.
      * Open the Terminal in your Compute Instance and clone your Git repository.

### **Phase 1: Data Ingestion & Vector Indexing**

This phase involves creating a repeatable pipeline to process and index your PDFs.

1.  **Create PDF Storage:**
      * In the Azure Portal, go to the storage account *associated with your Azure ML Workspace*.
      * Create a new **Blob Container** named `pdf-corpus`.
      * Upload your TCC-related PDF articles to this container.
2.  **Register Datastore in Azure ML:**
      * In Azure ML Studio, go to the "Data" section.
      * Create a new **Datastore** named `pdf_datastore` that points to your `pdf-corpus` blob container. This makes your PDFs easily accessible from any Azure ML job or notebook.
3.  **Create Azure AI Search Index:**
      * In the Azure AI Search portal, define a new **Index**.
      * Your index schema *must* include fields for the content and the vector.
      * **Example Schema:**
          * `id` (String, Key)
          * `content` (String, Searchable, Retrievable)
          * `source_file` (String, Filterable, Retrievable)
          * `content_vector` (Collection(Edm.Single), Vector Searchable, dimension: `1536` for `text-embedding-ada-002`)
      * Define a **Vector Search Profile** for the `content_vector` field (e.g., using `hnsw` algorithm).
4.  **Build the Indexing Job (Azure ML Job):**
      * This is the core data processing step. You will create a Python script and run it as an **Azure ML Command Job**.
      * In your Compute Instance, create a Python script (`run_indexing.py`):
          * **Connect:** Use `azure.identity.DefaultAzureCredential` to authenticate to all services.
          * **Load Data:** Use the Azure ML SDK (`ml_client.data.get()`) to mount or download data from your `pdf_datastore`.
          * **Process PDFs:** Use a library like `pypdf` or `PyMuPDF` to iterate through each PDF, extract text, and clean it.
          * **Chunk Text:** Use a text splitter (e.g., `langchain.text_splitter.RecursiveCharacterTextSplitter`) to break the text into small, overlapping chunks.
          * **Embed Chunks:** Create an `AzureOpenAIEmbeddings` client. Loop through your chunks and send them to your deployed `text-embedding-ada-002` model to get the vector embeddings.
          * **Push to Index:** Create an `azure.search.documents.SearchClient`. Batch-upload the documents (id, content, source\_file, content\_vector) to your Azure AI Search index.
      * Run this script as a reusable **Azure ML Job** so you can easily re-index when you add new PDFs.

### **Phase 2: RAG Pipeline Development (Prompt Flow)**

This is where you build the "brains" of the operation using Azure ML's visual RAG orchestrator.

1.  **Deploy Azure OpenAI Models:**
      * In your Azure OpenAI Studio, go to "Deployments."
      * Create two deployments:
        1.  **Embedding Model:** `text-embedding-ada-002` (e.g., deployment name `ada-embed`)
        2.  **Chat Model:** `gpt-35-turbo-16k` or `gpt-4` (e.g., deployment name `gpt-chat`)
2.  **Create Azure ML Connections:**
      * In Azure ML Studio, go to "Prompt Flow" -\> "Connections."
      * Create connections to your **Azure OpenAI** service and your **Azure AI Search** service. This securely stores the API keys and endpoints for Prompt Flow to use.
3.  **Build the Prompt Flow:**
      * Create a new **Chat Flow** in the Prompt Flow designer.
      * **Inputs:** The flow will automatically have `question` and `chat_history` as inputs.
      * **Flow Nodes (The RAG Pipeline):**
        1.  **`Embed_Question` (Python Tool):**
              * Takes the user's `question` as input.
              * Uses the **Azure OpenAI Connection** to call your `ada-embed` deployment.
              * Returns the `question_vector`.
        2.  **`Retrieve_Documents` (Python Tool):**
              * Takes the `question_vector` as input.
              * Uses the **Azure AI Search Connection** to perform a vector similarity search on your index.
              * Retrieves the `top-k` (e.g., k=4) most relevant `content` chunks.
              * Returns the retrieved chunks as a single string called `context`.
        3.  **`Format_Prompt` (Jinja Tool):**
              * This is your prompt engineering step.
              * It takes `question` and `context` as inputs.
              * **Template:**
                ```jinja
                system:
                You are a helpful AI assistant for a Software Engineering student. Answer the user's question based *only* on the context provided.
                If the information is not in the context, say "I could not find information about that in the provided documents."
                Do not make up answers. Cite the source file if possible.

                Context:
                {{context}}

                user:
                {{question}}
                ```
        4.  **`Chat_LLM` (LLM Tool):**
              * This node uses the **Azure OpenAI Connection** for your `gpt-chat` deployment.
              * It takes the formatted prompt from `Format_Prompt` as input.
              * It also takes the `chat_history` to enable follow-up questions.
      * **Outputs:** The flow's output will be the `answer` from the `Chat_LLM` node.
4.  **Test and Iterate:**
      * Use the Prompt Flow UI to test the chat flow with sample questions.
      * Adjust the Jinja prompt, the `top-k` value, or the chunking strategy (by re-running the Phase 1 Job) until you get high-quality, grounded answers.

### **Phase 3: Deployment & Application UI**

Now you make your RAG pipeline available as an API and build a UI for it.

1.  **Deploy Prompt Flow as an Endpoint:**
      * In the Prompt Flow UI, click the "Deploy" button.
      * Choose to deploy to a **Serverless Online Endpoint** (recommended for "pay-as-you-go" and auto-scaling) or a Managed Online Endpoint.
      * Azure ML will package your flow, create a container, and host it as a REST API.
      * Once deployed, go to the **Endpoints** tab in Azure ML Studio, find your endpoint, and copy its **REST Endpoint URL** and **Primary Key**.
2.  **Build the Frontend Chat Application:**
      * This will be a separate application that *consumes* your Azure ML endpoint.
      * Create a new **Azure App Service** (e.g., Python on Linux).
      * Develop a simple web app using **Streamlit** or **Flask/FastAPI**.
      * **App Logic (e.g., in Streamlit):**
          * Create a chat UI (`st.chat_input`, `st.chat_message`).
          * Store chat history in `st.session_state`.
          * When a user sends a message, your Python backend will:
            1.  Format the request payload (JSON) with the `question` and `chat_history`.
            2.  Create the `Authorization: Bearer <API_KEY>` header.
            3.  Use the `requests` library to make a `POST` call to your Azure ML Endpoint URL.
            4.  Receive the JSON response, extract the `answer`, and display it in the chat UI.
3.  **Deploy the UI:**
      * Connect your Azure App Service to your Git repository for CI/CD.
      * Push your Streamlit/Flask app code to deploy it.
      * Your TCC project is now live and accessible via the App Service URL.

### **Phase 4: Evaluation & Refinement (TCC Requirement)**

For a TCC, proving your system works is key. Use Azure ML's built-in evaluation tools.

1.  **Create a Test Dataset:**
      * In Azure ML Studio, create a small dataset (JSONL) of test questions and their "ground truth" (ideal) answers based on your PDFs.
2.  **Run a Batch Evaluation:**
      * In Prompt Flow, select your flow and choose "Run evaluation."
      * Run your flow against the test dataset.
      * Azure ML will calculate AI-assisted metrics like:
          * **Groundedness:** Does the answer stick to the provided context?
          * **Relevance:** Is the answer relevant to the question?
          * **Coherence:** Is the answer well-written?
3.  **Analyze and Refine:**
      * Use the evaluation results to find weaknesses.
      * Is the context not being retrieved correctly? -\> **Refine Phase 1** (change chunk size) or **Phase 2** (change `top-k`).
      * Is the LLM ignoring the context? -\> **Refine Phase 2** (make the Jinja prompt stricter).
      * Include these evaluation metrics and refinement loops in your final TCC paper.
