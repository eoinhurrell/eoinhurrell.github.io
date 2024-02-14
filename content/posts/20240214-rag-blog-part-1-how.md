+++
title = "Retrieval-Augmented Generation Experiments (Part 1)"
author = ["Eoin H"]
publishDate = 2024-02-14T00:00:00+00:00
tags = ["post"]
draft = false
+++

I've been researching LLMs, trying to find practical ways I can use them for myself, in my work or at home. One area that interests me is as tool to better understand the notes I've taken, topics I frequently learn about and areas I might want to explore more. I'm not the first to explore this territory (https://x.com/fortelabs/status/1749861644245848361).

I'll be looking at using a local model and building the system myself. I'm going to detail how I've built a toy system to chat with my notes. I'll be using Ollama[^1] to serve an open-source model locally, and building the application using LangChain[^2] and StreamLit[^3].

This will end with an interface I can use to ask questions that an LLM will attempt to answer, using (and even citing) notes I've made in my past reading. I hope to use this to assist in my learning on a number of topics I'm interested in, and in future posts I may evaluate how successfully this system aids in that task.

# Retrieval-Augmented Generation

Retrieval-Augmented Generation (RAG) is a revolutionary concept in conversational AI that combines data retrieval and generation models to create more comprehensive and accurate responses. Unlike traditional models relying solely on internal knowledge, RAG systems utilize external information from a pre-built index, enhancing the overall conversational experience. RAG will form the basis of my approach here, as it is a quick and easy way to provide context not available to the model during training. Alternatives would be either fine-tuning or training a model from scratch.

# Semantic Search

Semantic search is an essential component in building such an index. It uses vectors, mathematical representations of data points, to measure similarity between query and documents in a high-dimensional space. By converting textual information into numerical vectors, semantic search algorithms can efficiently analyze large datasets and retrieve the most relevant pieces of information.

In prior roles I have built semantic search using tools like ElasticSearch[^4], Spotify's annoy KNN library[^5] and similar tools. Vector store tooling has developed hugely as a result of the importance of embeddings in LLM work, so there's much less friction to throwing something together for a prototype. To get started, let's build our index:

```python
#!/usr/bin/env python3

from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langchain_community.embeddings import OllamaEmbeddings
from langchain_community.document_loaders import ObsidianLoader

loader = ObsidianLoader("<path to obsidian vault>")
docs = loader.load()

# print(docs[^0])
text_splitter = RecursiveCharacterTextSplitter()
documents = text_splitter.split_documents(docs)

vectorstore = FAISS.from_documents(documents, embedding=OllamaEmbeddings())
vectorstore.save_local("faiss_index")
```

This will create a local directory named 'faiss_index' that contains the vector store.

# StreamLit chatbot interface

Now that we have our index, let's explore how a RAG chatbot would work: When a user interacts with the bot, their query is first processed by a semantic search algorithm to find the most relevant documents from the index. These documents are then fed into a generation model along with the user's query to produce a response.

I've wrapped this flow into a StreamLit interface. This is adapted from a StreamLit tutorial on writing an LLM bot[^4], mostly to use Ollama. I've also added memory to the chatbot, so it will remember the line of questioning. I'm including all of the code here, as it's short and straightforward. Let's engage in conversation with our RAG-assisted chatbot using the index we built earlier:

```python
#!/usr/bin/env python3

from langchain_community.chat_models import ChatOllama
from langchain_community.embeddings import OllamaEmbeddings
from langchain.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain.schema import format_document
from langchain_core.messages import get_buffer_string
from langchain_core.runnables import RunnableLambda, RunnablePassthrough
from langchain_community.vectorstores import FAISS
from operator import itemgetter

from langchain.memory import ConversationBufferMemory
import streamlit as st


vectorstore = FAISS.load_local(
    "faiss_index", embeddings=OllamaEmbeddings())
retriever = vectorstore.as_retriever()
chat = ChatOllama(model="mistral")

memory = ConversationBufferMemory(
    return_messages=True, output_key="answer", input_key="question"
)
# First we add a step to load memory
# This adds a "memory" key to the input object
loaded_memory = RunnablePassthrough.assign(
    chat_history=RunnableLambda(
        memory.load_memory_variables) | itemgetter("history"),
)
DEFAULT_DOCUMENT_PROMPT = PromptTemplate.from_template(
    template="{page_content}")


def _combine_documents(
    docs, document_prompt=DEFAULT_DOCUMENT_PROMPT, document_separator="\n\n"
):
    doc_strings = [format_document(doc, document_prompt) for doc in docs]
    return document_separator.join(doc_strings)


_template = """Given the following conversation and a follow up question, rephrase the follow up question to be a standalone question, in its original language.

Chat History:
{chat_history}
Follow Up Input: {question}
Standalone question:"""
CONDENSE_QUESTION_PROMPT = PromptTemplate.from_template(_template)

template = """Answer the question based only on the following context:
{context}

Question: {question}
"""
ANSWER_PROMPT = ChatPromptTemplate.from_template(template)

# Now we calculate the standalone question
standalone_question = {
    "standalone_question": {
        "question": lambda x: x["question"],
        "chat_history": lambda x: get_buffer_string(x["chat_history"]),
    }
    | CONDENSE_QUESTION_PROMPT
    | chat
    | StrOutputParser(),
}

# Now we retrieve the documents
retrieved_documents = {
    "docs": itemgetter("standalone_question") | retriever,
    "question": lambda x: x["standalone_question"],
}
# Now we construct the inputs for the final prompt
final_inputs = {
    "context": lambda x: _combine_documents(x["docs"]),
    "question": itemgetter("question"),
}
# And finally, we do the part that returns the answers
answer = {
    "answer": final_inputs | ANSWER_PROMPT | chat,
    "docs": itemgetter("docs"),
}
# And now we put it all together!
final_chain = loaded_memory | standalone_question | retrieved_documents | answer

st.title("NoteChat")
# Set a default model
if "model" not in st.session_state:
    st.session_state["model"] = final_chain

# Initialize chat history
if "memory" not in st.session_state:
    st.session_state["memory"] = memory

if "messages" not in st.session_state:
    st.session_state["messages"] = []

# Display chat messages from history on app rerun
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# Accept user input
if prompt := st.chat_input("What is up?"):
    # Add user message to chat history
    st.session_state.messages.append({"role": "user", "content": prompt})
    # Display user message in chat message container
    with st.chat_message("user"):
        st.markdown(prompt)
    # Display assistant response in chat message container
    with st.chat_message("assistant"):
        message_placeholder = st.empty()
        full_response = ""
    response = final_chain.invoke({"question": prompt})
    memory.save_context({"question": prompt}, {
                        "answer": response["answer"].content})
    full_response = f"""{response['answer'].content}"""
    # for doc in response["answer"]["doc"]:
    #     full_response += f"\n- {doc['path']}"
    full_response += "\n"
    st.session_state.messages.append(
        {"role": "assistant", "content": full_response})
```

# Conclusion

The combination of RAG and semantic search using vector-based indexing allows conversational AI systems to deliver more accurate and comprehensive responses. With open-source tools like LangChain and Ollama, developers can easily experiment with these technologies and push the boundaries of conversational AI applications.

As a preliminary qualitative evaluation, it has been interesting to explore how an LLM 'understands' my notes. I'm impressed by how it will draw in contrasting ideas I wouldn't have thought of, but it also has issues, it frequently gets confused and doesn't respond as I would expect, potentially because the corpus of my notes is very diverse, it includes highlights on anything I have read. I've seen the LLM respond as a medieval lord, because it pulled in a note from a fantasy book when I asked a philosophical question. Further work is needed to tune the prompt I'm using. I'd also like to find ways to do more quantitative evaluation of the results returned, even though this is only a toy side project.

### Footnotes

[^1]: Ollama is a great open-source local LLM server https://ollama.com
[^2]: LangChain is an open-source framework designed for developing applications using LLMs. https://langchain.ai
[^3]: StreamLit is a powerful way to turn data scripts into shareable web apps https://streamlit.io
[^4]: https://www.elastic.co/elasticsearch, or the open alternative https://opensearch.org
[^5]: https://github.com/spotify/annoy
[^6]: https://docs.streamlit.io/knowledge-base/tutorials/build-conversational-apps
