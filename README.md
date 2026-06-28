Ask a Bill: RAG Pipeline Over Federal Legislation
I built a system that lets you ask plain-English questions about a real piece of federal legislation and get answers pulled straight from the bill's text, not from a model's memory. It runs in n8n, embeds documents locally with Ollama, stores them in a vector database, and uses Claude to write the final answer.

The point is simple: nobody reads a 567-page appropriations bill. This lets you ask it a question instead.

The whole project cost 6 cents in API spend to build and test.

Screenshots: 
<img width="1664" height="854" alt="Screenshot 2026-06-28 152802" src="https://github.com/user-attachments/assets/a8da108b-b62c-4be8-ba2a-64a566c15bfa" />
<img width="1683" height="286" alt="Screenshot 2026-06-28 153014" src="https://github.com/user-attachments/assets/7dddc929-d718-4747-952f-a490b3ed2d12" />

The problem
A large language model only knows what it saw in training. Ask it about a specific document it has never seen, like a bill passed last month, and it does one of two bad things: admits it does not know, or worse, confidently hallucinates. For legislation, where an exact dollar figure or the exact list of covered items is the whole point, a confident wrong answer is useless and pretty dangerous.

The naive fix is to paste the entire document into the prompt every time but that breaks fast. These documents are huge, sending all that text on every question is wasteful, and the model actually gets worse at finding the relevant detail when it is buried in hundreds (or thousands) of pages.

Retrieval Augmented Generation(RAG), is a better fix. Before the model answers, the system retrieves only the few most relevant pieces of the document, adds just those pieces to the prompt, and lets the model generate an answer grounded in them. Instead of "answer from memory" or "here’s everything, good luck," it becomes "here’s the three paragraphs that actually relate to this question, answer using these."
The document
I used H.R. 7148, the Consolidated Appropriations Act, 2026, signed into law as Public Law 119-75. It’s one of this year's government funding bills, the kind of dense, consequential document that almost cannot be read end to end in a productive amount of time. Federal legislation is public domain, so it is free to index and publish. For this build I indexed a forgiving slice of the bill rather than all 567 pages, enough to prove the system works before scaling up.
How it works
The system runs in two phases.

Phase 1, indexing (done once):

Document -> split into chunks -> embed each chunk (Ollama) -> store in vector database

Phase 2, querying (per question):

Question -> embed the question (Ollama) -> find nearest chunks in the store -> add chunks to prompt -> Claude answers

The key idea is that the expensive understanding happens once, at indexing time. Each chunk of text gets turned into an embedding, a list of numbers that captures its meaning as a point in space (768 numbers, in this case). Chunks with similar meaning land near each other, even when they share no actual words. So at question time, the system just embeds the question into the same kind of point and asks the database which stored chunks sit closest. That is a fast distance lookup, not a re-read of the whole document, which is what makes it scale.
The decisions that actually matter
Local embeddings, zero cost. Instead of a paid embedding API, I ran the embedding model (nomic-embed-text) locally through Ollama. No API key, no per-call cost. This is the entire reason the project cost 6 cents instead of dollars: the only paid step is the final answer generation. It also shows the embedding model does not have to be a cloud service.

Cheap, free local model for embeddings; Claude only for the answer. Embedding runs locally and free on every chunk and every question. Claude Sonnet 4.6 only runs once per question, to write the final answer a person reads. Matching the tool to the job keeps cost near zero.

The question must be embedded with the same model as the documents. Retrieval compares the question's coordinate against the documents' coordinates. If they were produced by different embedding models they would live in different number-spaces and the comparison would be meaningless. Same model in, comparable coordinates out.

I’ll never trust "it ran" over accuracy. A RAG system can return a confident, complete-sounding answer that is actually missing half the information, because retrieval sneakily grabbed the wrong or incomplete chunks. The only way to catch that is to ask questions you know the answer to and check the output against the source. That is exactly how I found the bug below.
The bug, and the controlled fix
I tested the system with a question whose full answer I knew: which missiles and systems are approved for multiyear procurement, and for how long. The correct answer has two parts, a five-year list and a seven-year list.

The system returned the five-year list perfectly and silently dropped the seven-year list. Worse, it gave no sign anything was missing. It read like a complete answer (and I totally would’ve bought it!). That’s a dangerous failure mode of RAG: partial retrieval producing a confident, incomplete answer.

I had two levers to fix it: make the chunks bigger so the whole section stays together, or retrieve more chunks so the missing piece gets pulled in. My first instinct, and my initial assumption, was that chunk size was the cause. So I raised the chunk size and re-tested. The answer was unchanged.

Instead of guessing, I ran a controlled test: I held chunk size constant and changed only the retrieval depth (Top K), the number of chunks pulled per question.

Top K 4: only the five-year list.
Top K 8: both lists, complete.

That isolated it. The fix was retrieval depth, not chunk size. The reason is subtle and but worth stating: the chunk containing the answer existed in the index the whole time, but retrieval ranks chunks by how similar they are to the question, not by whether they contain the answer. The right chunk was ranking just below the default cutoff of 4, so it never made it into the prompt. Raising the cutoff to 8 surfaced it.

The lesson I took from this: "the answer is in a chunk" is not the same as "that chunk ranks highly for the question." And changing two variables at once tells you nothing about which one mattered. Changing one at a time told me exactly what was wrong.
Why I did not just crank the setting up
Once I knew a higher Top K worked, the lazy move would be to set it very high to be safe. I did not, because more retrieved chunks is not strictly better. Every extra chunk is one ranked lower, meaning less relevant, so pushing the number too high pads the prompt with noise that can make the answer worse and costs more tokens on every query. Top K 8 was enough to reliably catch the right chunk without dragging in junk. That number is right for this small index; a much larger document would need re-tuning, because catching the right chunk out of thousands is a different problem than out of eleven.
Final settings
Chunk size: 600 characters
Chunk overlap: 40 characters
Retrieval depth (Top K): 8
Embeddings: nomic-embed-text, run locally via Ollama
Generation: Claude Sonnet 4.6
Limits and what's next
The vector store is in-memory. It only holds data during a single run, which means indexing and querying have to happen in one pass. That is fine for a demo but not for anything real. The next step is a persistent vector store so the document is indexed once and queried any time after.
It runs locally, not as a live service. Right now it runs on my machine through n8n. Turning it into a public chatbot people can actually use means hosting it, exposing it safely, indexing the full bill, and adding cost and abuse controls. That is a separate, larger project.
Indexed a slice, not the whole bill. The current index covers a representative portion of H.R. 7148. Scaling to all 567 pages is straightforward but would require re-tuning chunk size and retrieval depth for the larger volume.
Stack
n8n (self hosted with Docker), Ollama running nomic-embed-text locally for embeddings, an in-memory vector store, and Claude Sonnet 4.6 for answer generation. The workflow is exported as JSON in this repo, so anyone can import it and run it with their own free local Ollama and their own API key.

As always, thanks for the read!
-Michael Swayney
