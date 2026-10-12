# Hybrid Menu Architecture: The "Best of Both Worlds"

> **Decision**: You should **NOT** choose between "Pure RAG" and "Pure SQL".
> **Solution**: Use **Hybrid Search** (SQL Table with a Vector Column).

## 1. The Bottleneck You Identified

You are absolutely right:

- **Pure RAG (PDF/CSV)**: Bad for inventory. If "Lobster" goes out of stock at 8 PM, the AI reads the PDF and still sells it. Result: Angry guest.
- **Pure SQL**: Bad for understanding. If user asks "something spicy", a SQL query `SELECT * FROM menu WHERE name LIKE '%spicy%'` fails if the dish is named "Chili Chicken" (doesn't contain the word "spicy").

## 2. The Solution: Structured RAG (Hybrid)

We keep the data in **Supabase (Postgres)**, but we treat it as **both**.

### The Schema

Instead of a "Text Chunk" from a PDF, we have a proper Row.

| ID  | Name            | Description                           | Price | **is_available** (SQL) | **embedding** (RAG) |
| :-- | :-------------- | :------------------------------------ | :---- | :--------------------- | :------------------ |
| 1   | Volcano Chicken | Extremely hot chili peppers with rice | $20   | ✅ TRUE                | `[0.42, 0.91, ...]` |
| 2   | Mild Soup       | Gentle corn soup                      | $10   | ❌ FALSE               | `[0.11, 0.05, ...]` |

### The Workflow

**Step 1: Ingestion (One Time / Update)**
We write a script to read your `menu.csv`.

- It reads "Volcano Chicken", "Extremely hot...".
- It sends that text to OpenAI/Mistral to get an embedding.
- It inserts the Row + Embedding into Supabase.

**Step 2: The User Query (Runtime)**

- **User**: "I want something super spicy."
- **AI**: Generates vector for "something super spicy".
- **Database Query**:
  ```sql
  SELECT name, price, description
  FROM menu_items
  WHERE is_available = TRUE             -- 1. SQL Rule (Inventory Check)
  AND category != 'Drinks'              -- 2. SQL Rule (Business Logic)
  ORDER BY embedding <-> user_query_vector -- 3. Semantic Sort (RAG)
  LIMIT 3;
  ```

**Step 3: Managing Inventory**

- The Kitchen runs out of "Volcano Chicken".
- Kitchen Dashboard updates Supabase:
  ```sql
  UPDATE menu_items SET is_available = FALSE WHERE id = 1;
  ```
- **Result**: The NEXT semantic search immediately excludes it because of the `WHERE is_available = TRUE` clause. **Zero latency sync.**

## 3. Why this wins

1.  **Inventory Tracking**: Solved. It's real-time SQL.
2.  **Smart Understanding**: Solved. It finds "Volcano Chicken" even if the user says "Spicy", because the vector matches.
3.  **Scalability**: Postgres handles millions of rows easily.
4.  **No Hallucinations**: The AI is grounded in the database rows, not random text chunks from a PDF.

## 4. Implementation Plan for V2

1.  **Stop** defining the menu as a collection of text chunks.
2.  **Start** treating the menu as a **Database Table**.
3.  **Pipeline**:
    - `menu.csv` -> `ingestion_script.py` -> `Supabase Table`
4.  **Agent Logic**:
    - Agent Tool: `search_menu(query: str)` -> executes the Hybrid SQL query above.

This is the standard architecture for e-commerce and dynamic inventory RAG.
