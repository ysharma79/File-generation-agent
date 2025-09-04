# Complete Guide: Integrating a Dify File-Generation Agent with AnythingLLM

This guide provides a comprehensive walkthrough for creating a custom file-generation agent using Dify and integrating it seamlessly into AnythingLLM without modifying the core application code. 

We will cover three main stages:
1.  **Part 1**: Building the agent and its tools in Dify.
2.  **Part 2**: Hosting the necessary microservice on the cloud (e.g., Cloudflare).
3.  **Part 3**: Connecting the agent to AnythingLLM using its native custom agent feature.

---

## Part 1: Building the File-Generation Agent in Dify

First, we need to create an agent in Dify and give it a "tool" that can generate files. This tool will be a simple API endpoint exposed from our `file-generator` microservice.

### Step 1.1: Prepare the External Microservice

The `file-generator` microservice is a standalone project. Before proceeding, ensure you have moved its directory **outside** of the AnythingLLM project folder.

Once the `file-generator` directory is in a separate location, navigate into it and run the application:

```bash
# Navigate to wherever you moved the file-generator directory
cd /path/to/your/file-generator

# Build the Docker image
docker build -t file-generator .

# Run the container, exposing port 5001
docker run -d -p 5001:5001 --name file-generator-container --rm file-generator
```

### Step 1.2: Expose the Microservice to the Internet

Dify is a cloud service, so it needs a public URL to access our local microservice. We'll use `ngrok` to create a secure tunnel.

1.  [Download and install ngrok](https://ngrok.com/download).
2.  Run the following command to expose port 5001:
    ```bash
    ngrok http 5001
    ```
3.  Ngrok will provide a public `Forwarding` URL (e.g., `https://<unique-id>.ngrok.io`). **Copy this URL**, as we'll need it in the next step.

### Step 1.3: Create a Custom Tool in Dify

Now, let's create the Dify agent and teach it how to use our microservice.

1.  Log in to your Dify account and create a new application, selecting **Agent** as the type.
2.  Navigate to the **Tools** section from the left sidebar.
3.  Click **Add Tool** and select **Create Custom Tool**.
4.  Configure the tool as follows:
    *   **Tool Name**: `create_file`
    *   **Tool Description**: `Generates a file of a specified type (pdf, docx, xlsx) with given content and a filename.`
    *   **API Endpoint**: Set the method to `POST` and enter the URL: `https://<your-ngrok-id>.ngrok.io/generate/{file_type}`.
    *   **API Parameters**: 
        1.  Under **Path**, Dify will automatically detect `file_type`. Mark it as **Required**.
        2.  Switch to the **Body** tab, select `application/json`, and add two properties:
            *   `filename` (string, required)
            *   `content` (string, required)
5.  Save the tool.

### Step 1.4: Configure the Dify Agent's Prompt

In your Dify agent's **Prompt** section, you must define the context variables that our microservice will send. Go to the **Context** section (usually above the prompt), click **Add**, and add two variables:

1.  `last_assistant_message` (string)
2.  `formatted_chat_history` (string)

Now, paste the following instructions into the **Prompt** box. This prompt teaches the agent how to prioritize which content to use.

**Final Advanced Prompt:**

```text
You are a highly intelligent file generation assistant with three functions. You must follow a strict order of priority to decide which content to use for the file.

**Priority 1: Save the Last Assistant Response**

If the user's prompt includes keywords like "last response", "that response", "the last message", or similar phrases, you MUST use the content from the `{{#inputs.last_assistant_message#}}` variable.
-   **Example Trigger**: "Save the last response as a PDF."
-   **Action**: Use the content of `last_assistant_message` for the `create_file` tool.

**Priority 2: Save the Full Conversation History**

If the user's prompt does NOT ask for the last response but includes keywords like "this conversation", "the whole chat", or "chat history", you MUST use the content from the `{{#inputs.formatted_chat_history#}}` variable.
-   **Example Trigger**: "Save this conversation as a DOCX."
-   **Action**: Use the content of `formatted_chat_history` for the `create_file` tool.

**Priority 3: Create a File from Prompt Content**

If neither of the above conditions is met, the user is providing the content directly. You must extract the `file_type`, `filename`, and `content` from the user's prompt.
-   **Example Trigger**: "Create a file named report.pdf with the content 'Hello World'."
-   **Action**: Use the content you extract from the prompt for the `create_file` tool.

After successfully calling the tool for any function, respond with a confirmation and a markdown link to the file URL provided by the tool.
```

---

## Part 2: Hosting the Bridge Microservice on Cloudflare

For a production setup, you need a permanent public URL. Hosting the `file-generator` microservice on a cloud platform like Cloudflare is a great option.

### Step 2.1: Push the Docker Image to a Registry

Cloud providers need to pull your Docker image from a registry.

1.  **Tag your image**: 
    ```bash
    docker tag file-generator <your-docker-hub-username>/file-generator:latest
    ```
2.  **Push the image**:
    ```bash
    docker push <your-docker-hub-username>/file-generator:latest
    ```

### Step 2.2: Deploy on a Cloud Service

Deploying a container can be done on many platforms (Cloudflare, AWS, GCP, DigitalOcean). The general steps are:

1.  Choose a container hosting service (e.g., Cloudflare Workers, DigitalOcean App Platform, AWS App Runner).
2.  Create a new service and point it to the Docker image you just pushed.
3.  Configure the service to listen on port `5001`.
4.  The platform will provide a permanent public URL (e.g., `https://my-file-gen.my-provider.com`). **Copy this URL.**

**Note**: This is a high-level overview. Each cloud provider has specific documentation for deploying containers.

---

## Part 3: Integrating with AnythingLLM (No Code Change)

Finally, let's connect our hosted agent to AnythingLLM.

### Step 3.1: Add the Custom Agent in AnythingLLM

1.  Open your AnythingLLM application.
2.  Go to **Settings** > **Custom Agents**.
3.  Click **Import Agent**.
4.  In the "Import from URL" field, paste the public URL of your **hosted microservice** from Part 2, followed by `/anythingllm-manifest.json`.
    *   **Example**: `https://my-file-gen.my-provider.com/anythingllm-manifest.json`
5.  Click **Import**.

### Step 3.2: Configure and Enable the Agent

1.  The "Dify File Generator" will now appear in your list of custom agents. Click it.
2.  Fill in the required **Setup Arguments**:
    *   `Dify API Endpoint`: Your Dify application's API endpoint.
    *   `Dify API Secret Key`: Your Dify application's secret key.
3.  Save the changes.
4.  Navigate to the workspace where you want to use the agent, go to **Settings** > **Agent Skills**, and toggle on the "Dify File Generator".

### Step 3.3: Test the End-to-End Flow

Go to the workspace chat and test the agent's capabilities with a few prompts:

**Test Case 1: Create a file from content**

`Create a DOCX file named 'report.docx' with the content: This is the first quarterly report.`

**Test Case 2: Save the conversation history**

After a few exchanges with the agent, ask it to save the chat:

`Now, save this entire conversation as a PDF named 'chat_log.pdf'.`

**Test Case 3: Save the last assistant response**

First, ask the agent or another LLM to generate some content. For example: "Tell me about the history of the internet." After it responds, give the following command:

`Save that last response as a file named 'internet_history.docx'.`

AnythingLLM will now use its new skill to call your microservice, which will invoke the Dify agent, generate the file, and return a download link directly in the chat. You have successfully extended AnythingLLM's functionality without touching its source code.
