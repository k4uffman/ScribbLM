# ScribbLM - AI-Powered Educational Whiteboard

**Live Application & Demo:** [https://www.scribblm.com](https://www.scribblm.com)

---

> **Note on Source Code:** The source code for ScribbLM is currently private due to its proprietary nature and ongoing commercialization discussions. This repository serves as a comprehensive technical overview of the project's architecture, features, and the engineering challenges involved. I am happy to provide a live, in-depth code walkthrough upon request.

## Project Overview

ScribbLM is a full-stack, AI-powered, real-time collaborative whiteboard designed for lifelong learners. It moves beyond simple note-taking by integrating a context-aware AI assistant directly into a freeform digital canvas. The goal is to create a dynamic and personalized learning environment where users can explore concepts visually and receive intelligent guidance.

## Core Features & Technical Implementation

### 1. Context-Aware AI Assistant
* **Functionality:** Users can ask questions, request explanations, or ask the AI to "visualize" a concept. The assistant can analyze user-selected portions of the whiteboard or the entire canvas for context.
* **Implementation:**
    * Leveraged a multi-model strategy using **Google's Gemini 2.0 Flash** for conversational Q&A and **Anthropic's Claude 3.5 Sonnet** for SVG visualization generation.
    * Frontend captures user prompts and whiteboard state (as a PNG data URL), sending them to a Next.js Server Action.
    * User preferences (learning style, complexity) are fetched from the database and injected into the system prompt to personalize the AI's responses.

### 2. Semantic Search
* **Functionality:** Allows users to search across all their notes using natural language queries, finding results based on conceptual meaning rather than exact keyword matches.
* **Implementation:**
    * Utilized **Google's `text-embedding-004` model** to generate 768-dimension vector embeddings for each note.
    * Embeddings are created from a composite of the note's title, description, extracted text from the whiteboard, chat history, and an AI-generated description of the visual elements.
    * Stored embeddings in a **Supabase (Postgres) table with the `pgvector` extension**.
    * When a user searches, their query is converted into an embedding, and a `match_notes` RPC function is called to perform a cosine similarity search against the stored vectors.

### 3. Real-time Collaborative Whiteboard
* **Functionality:** A freeform, infinite canvas where users can draw, write, code, and connect ideas. Changes are saved automatically in real-time.
* **Implementation:**
    * Built using the **tldraw** library, providing a feature-rich whiteboard experience.
    * Whiteboard state is captured as a `TLStoreSnapshot` (JSON object) and saved to the Supabase database.
    * Implemented a debounced save function to efficiently persist changes to the backend without overwhelming the database, with an immediate save on page exit to prevent data loss.

## Tech Stack & Architecture

| Category           | Technology                                          |
| ------------------ | --------------------------------------------------- |
| **Frontend** | Next.js (App Router), React, TypeScript, Tailwind CSS |
| **Backend** | Next.js (Server Actions)                            |
| **Database** | Supabase (Postgres, Auth, Storage, `pgvector`)      |
| **AI / ML** | Google Gemini, Anthropic Claude, Google Embeddings  |
| **Whiteboard** | tldraw                                              |
| **UI Components** | Radix UI, Shadcn UI                                 |

*(A high-level architecture diagram would show: Client (Next.js) -> Supabase (Auth, DB, Vector Store) & Next.js Server Actions -> Google/Anthropic APIs)*

---

## Code Highlights

Below are selected snippets that demonstrate key architectural patterns and features of the application.

### 1. Generating Rich Embeddings for Semantic Search
This Next.js Server Action shows the end-to-end pipeline for creating a semantic vector for a note. It gathers text from multiple sources, uses an AI vision model to describe the canvas, generates an embedding with Google's API, and upserts it into the `pgvector` enabled database.

```typescript
// app/actions/embedding.ts

'use server'

import { createSupabaseServerClient } from '@/lib/supabase/server'
import { GoogleGenerativeAI } from '@google/generative-ai'
import { TLStoreSnapshot, TLRecord, TLShape } from 'tldraw'
import { getDescriptionFromImage } from './gemini'

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_API_KEY!)

// ... (helper functions) ...

export async function generateAndStoreEmbedding(noteId: string, whiteboardImageDataUrl?: string) {
  const supabase = await createSupabaseServerClient()
  try {
    // 1. Fetch all data sources for the note
    const { data: note } = await supabase.from('notes').select('id, title, description, whiteboard_data').eq('id', noteId).single()
    const { data: chatMessages } = await supabase.from('chat_messages').select('content').eq('note_id', noteId)

    // 2. Get AI vision description of the whiteboard image
    let visionDescription = '';
    if (whiteboardImageDataUrl) {
      const visionResult = await getDescriptionFromImage(whiteboardImageDataUrl);
      if (visionResult.success && visionResult.description) visionDescription = visionResult.description;
    }

    // 3. Combine all text into a single document for embedding
    const textToEmbed = [
      note.title,
      note.description,
      extractTextFromWhiteboard(note.whiteboard_data),
      chatMessages ? chatMessages.map(m => m.content).join(' ') : '',
      visionDescription
    ].filter(Boolean).join(' ').trim();

    if (!textToEmbed) return;

    // 4. Generate the embedding using Google's model
    const model = genAI.getGenerativeModel({ model: 'text-embedding-004' })
    const result = await model.embedContent(textToEmbed)
    const embedding = result.embedding.values

    // 5. Store the vector in the Supabase pgvector table
    await supabase.from('note_embeddings').upsert({
      note_id: noteId,
      embedding: embedding,
      updated_at: new Date().toISOString()
    });

  } catch (error) {
    console.error('Error in generateAndStoreEmbedding:', error)
  }
}
```

### 2. Dynamic & Personalized AI System Prompts
This server action demonstrates how user-specific data is used to tailor the AI's behavior. It fetches the user's learning preferences and dynamically constructs a `system` prompt, ensuring the Gemini model's response is personalized.

```typescript
// app/actions/gemini.ts

'use server'

import { getUserPreferences } from './user'

export async function sendMessageToGemini(messages: Message[]) {
  try {
    // 1. Fetch the current user's learning preferences from the database.
    const userPreferences = await getUserPreferences();

    // 2. Construct a dynamic system prompt based on those preferences.
    let baseSystemPrompt = `You are ScribbLM Assistant, a friendly and concise AI designed for a whiteboard application...`;

    if (userPreferences) {
        baseSystemPrompt += `

        Please tailor your response to the user's preferences:
        - Learning Style: ${userPreferences.learning_style}
        - Feedback Preference: ${userPreferences.feedback_preference}
        - Complexity Level: ${userPreferences.complexity_level}
    `;
    }

    const systemInstruction = { role: 'system', content: baseSystemPrompt }

    // 3. Prepend the system instruction and execute the API call to the Gemini model.
    const contents = [systemInstruction, ...messages].map(/* ... */);
    const response = await fetch(/* ... */);

    // ...
  } catch (error) {
    // ...
  }
}
```

### 3. Debounced & Eager Saving on the Frontend
This React component demonstrates a robust saving mechanism for the real-time whiteboard. It uses a `debounce` utility to batch frequent updates but also leverages a `useEffect` cleanup function to trigger an immediate save when the user navigates away, preventing data loss.

```typescript
// app/whiteboard/[noteId]/page.tsx

// ... (imports and debounce implementation) ...

export default function WhiteboardPage() {
  const editorRef = useRef<Editor | null>(null);

  // An immediate save function
  const immediateSave = useCallback(async (editor: Editor) => {
    // ... (implementation to save snapshot and trigger embedding) ...
  }, [noteId, supabase]);

  // A debounced version for frequent changes
  const debouncedSave = useMemo(
    () => debounce((editor: Editor) => immediateSave(editor), 500),
    [immediateSave]
  );

  // Effect to ensure data is saved when the component unmounts
  useEffect(() => {
    return () => {
      if (editorRef.current) {
        immediateSave(editorRef.current);
      }
    };
  }, [immediateSave]);

  return (
    <Tldraw
      onMount={(editor: Editor) => {
        editorRef.current = editor;
        // Listen for user changes and trigger the debounced save
        editor.store.listen(() => {
            if (editorRef.current) debouncedSave(editorRef.current);
        }, { source: 'user', scope: 'document' });
      }}
    />
  );
}
```

## Next Steps

I am currently exploring potential institutional partnerships for ScribbLM. I am also actively seeking full-time roles where I can apply my experience in building and deploying end-to-end AI solutions.

**For a detailed discussion or a live code walkthrough, please contact me at kauffmand12@gmail.com.**