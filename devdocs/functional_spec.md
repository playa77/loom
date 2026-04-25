**Functional Requirements Document (FRD)**

**I. Executive Summary**
This application is a privacy-focused, local-first desktop client designed to provide users with seamless access to a wide variety of external AI models via a centralized routing service. The core value proposition is offering a highly customizable, multi-workspace environment where users can manage complex, long-running AI conversations without risking data loss. It solves the problem of fragmented AI interactions by allowing users to curate persistent personal preferences and project-specific knowledge bases, which are silently injected into conversations, ensuring the AI always operates with the full context of the user's intent and standards without cluttering the visible chat interface.

**II. User Capabilities (The "What")**

*Workspace & Navigation Management*
*   The user must be able to open multiple independent workspaces (tabs) in order to engage in parallel conversations.
*   The user must be able to switch between open workspaces in order to resume different contexts instantly.
*   The user must be able to reorder open workspaces in order to organize their workflow.
*   The user must be able to close a workspace in order to declutter the interface.

*Conversation Management*
*   The user must be able to create a new conversation in order to start a fresh context.
*   The user must be able to load an existing conversation from a history list in order to resume past work.
*   The user must be able to rename a conversation in order to organize their history.
*   The user must be able to delete a conversation in order to remove it permanently.
*   The user must be able to export a conversation in order to back it up or share it.
*   The user must be able to import a conversation in order to restore past work.

*Chat Interaction*
*   The user must be able to type and send a message in order to prompt the AI.
*   The user must be able to stop an ongoing AI response in order to prevent unwanted generation.
*   The user must be able to edit their previously sent message and re-submit it in order to change the direction of the conversation.
*   The user must be able to regenerate an AI response in order to receive an alternative answer.
*   The user must be able to delete a specific message (and all subsequent messages) in order to prune the conversation history.
*   The user must be able to copy a single message as plain text, formatted text, or with hidden attachment data included in order to transfer context externally.
*   The user must be able to copy an entire conversation as plain text, formatted text, or with hidden attachment data included in order to archive the full context.

*File Handling*
*   The user must be able to select and attach local files (documents, code, archives) to their message in order to provide external context to the AI.
*   The user must be able to remove a staged file attachment before sending in order to correct mistakes.

*Project & Knowledge Management*
*   The user must be able to create, edit, and delete a "Project" in order to establish a curated knowledge vault.
*   The user must be able to add, edit, and delete raw text and file attachments to a Project's Knowledge base in order to define the project's permanent context.
*   The user must be able to assign an active conversation to a specific Project in order to associate the conversation with that project's knowledge.
*   The user must be able to view a clear indicator (badge/label) on the conversation interface in order to know which project the conversation belongs to.

*Personalization & Settings*
*   The user must be able to toggle the use of Personal Memories on or off in order to control whether persistent preferences are applied.
*   The user must be able to enter and save long-form text for Personal Memories in order to define their default communication style, constraints, and preferences.
*   The user must be able to select an AI model in order to dictate which engine processes their request.
*   The user must be able to adjust the response randomness (temperature) in order to control the creativity of the AI.
*   The user must be able to toggle web search capability in order to allow the AI to access external information.
*   The user must be able to select, create, edit, and delete System Prompts in order to define the AI's persona and behavioral rules.
*   The user must be able to customize font families and sizes for chat text, interface text, and code text in order to optimize readability.
*   The user must be able to toggle markdown rendering in order to view raw or formatted AI output.
*   The user must be able to toggle a close-confirmation prompt in order to prevent accidental application closure.
*   The user must be able to input and save their external service access key in order to authenticate with the AI routing service.

**III. Core Business Entities**

*   **Conversation**: Consists of a unique identifier, a title, a creation date, a sequence of Messages, and an optional association with a Project.
*   **Message**: Consists of a unique identifier, a role (User, Assistant, or System), the content text, the model used (if applicable), and the temperature used (if applicable).
*   **Project**: Consists of a unique identifier, a name, and a Knowledge base. Acts as a curated knowledge vault associated with one or more Conversations.
*   **Project Knowledge**: Consists of raw text notes and file attachments that form the persistent context of a Project.
*   **Personal Memory**: Consists of a single, long-form text block representing durable user preferences and defaults applied across all conversations when active.
*   **System Prompt**: Consists of a unique identifier, a name, the prompt text, and usage notes (internal to the user, not sent to the AI).
*   **Model Profile**: Consists of a display name and a specific model identifier used to route the request.
*   **File Attachment**: Consists of a file name, the original file path, and the extracted text content of the file.

**IV. Automated System Behaviors**

*   **Hidden Context Injection**: When a message is sent, the system must automatically prepend the active Personal Memories and the associated Project Knowledge to the request as hidden system context. The user must not see this injected text in their chat interface.
*   **File Content Extraction**: When a user attaches a document, the system must automatically extract all readable text while ignoring images and binary data. For archive files, the system must automatically list the contents and extract text from the individual text-based files within the archive.
*   **Attachment Display Masking**: The system must hide the raw extracted text of attached files from the visible chat bubbles, displaying only a compact visual indicator (badge) showing the file name.
*   **Automatic Conversation Naming**: When a user sends their first message in a new conversation, the system must automatically generate a title based on the text of that first message.
*   **Real-Time State Synchronization**: The system must instantly synchronize the application state across multiple open windows without data loss or visual flickering.
*   **Real-Time Response Streaming**: The system must display AI responses incrementally as they are generated, rather than waiting for the entire response to complete.
*   **Exhaustive Token Estimation**: The system must continuously calculate and display an estimated token count that includes the visible chat history, the pending user input, the active Personal Memories, and the associated Project Knowledge.
*   **High-Context UI Decoupling**: The system must decouple heavy processing operations from the main interface rendering in order to maintain exceptional clarity and responsiveness when handling massive context windows (250,000+ tokens).
*   **Legacy Data Migration**: Upon first launch, if legacy application data is detected in the user's home directory, the system must automatically copy it to the new data directory.
*   **Model Metadata Caching**: The system must periodically fetch available model capabilities from the external service and cache this information locally for up to 6 hours to minimize external calls.

**V. Business Rules, Limits, and Validations**

*   **Transactional Message Integrity (Zero-Loss)**: Message handling must be wrapped in transactional logic. Under no circumstances (including broken, rejected, or overloaded external service responses) can a user's message or the conversation thread disappear or be lost.
*   **Memory Character Limit**: Personal Memory text cannot exceed 28,000 characters. The system must enforce this hard cap during input.
*   **File Size Limits**: A single attached file cannot exceed 2MB of extracted text. A single file inside an archive cannot exceed 512KB of extracted text. The total extracted text from a single archive cannot exceed 2MB. PDF files cannot exceed 4MB in raw file size.
*   **Context Hierarchy**: If a direct user instruction in the current chat conflicts with stored Personal Memories or Project Knowledge, the system must instruct the AI to follow the current chat instruction.
*   **Memory Secrecy**: The system must instruct the AI not to reveal the contents of Personal Memories or Project Knowledge unless the user explicitly asks for them.
*   **Web Search Modifier**: If the user toggles web search on, the system must append a specific modifier to the selected model identifier before sending the request; if toggled off, this modifier must be removed.
*   **API Key Requirement**: The system must prevent chat interactions if the external service access key is not configured, and must prominently prompt the user to enter it.
*   **Fallback Extraction Strategy**: If a document's text cannot be extracted using the primary method, the system must attempt secondary extraction methods before returning a failure message to the user.
*   **Data Resilience**: If the application is unexpectedly closed, all conversation history up to the last sent message must be retained.
*   **Close Confirmation**: If the close confirmation setting is active, the system must require explicit user approval before terminating the application.

**VI. External Service Requirements**

*   **AI Provider Routing Service**: The system requires a secure connection to an external AI routing service to transmit the user's prompt, conversation history, attached document text, personal memories, and project knowledge. The service must support real-time streaming of the generated response.
*   **Model Metadata Service**: The system requires a connection to the routing service's public catalog to retrieve available model identifiers and capabilities.
*   **External Font Service**: The system requires access to an external web font service in order to dynamically load custom typography selected by the user for the interface, chat, and code displays.
