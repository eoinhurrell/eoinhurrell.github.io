+++
title = "Easy Similarity Search with Custom Vector Embeddings and Typesense"
author = ["Eoin H"]
publishDate = 2024-06-03T00:00:00+00:00
tags = ["post"]
draft = false
+++

I've been kicking the tires on [Typesense](https://typesense.org/), an open-source search engine and alternative to [Elasticsearch](https://www.elastic.co/elasticsearch), for a personal project. So far I'm quite impressed with it. This is a small post recording how easy it was to set up vector embedding fields using a custom model.

It is getting much easier to build semantic search applications. In previous work when I was setting up semantic search it was both exciting and frustrating, because the tooling wasn't there and you had to build basically every piece yourself. It was cutting-edge, but you had to make your own cutting tools before you really saw the benefit. In my case, I had been using Elasticsearch and had to augment it with [annoy](https://github.com/spotify/annoy), (using approximate K nearest neighbour in place of cosine distance, until Elasticsearch added vector search support).

## Using Custom Models

Typesense natively supports [vector search](https://typesense.org/docs/26.0/api/vector-search.html), and integrates with OpenAI, OpenAI-compatible APIs and even loading its own ONNX embedding models (it has a number of models it can use of this purpose). As I've been keeping up with embedding models, I've seen the work by [Mixed Bread AI](https://www.mixedbread.ai/blog/mxbai-embed-large-v1) on their `mxbai-embed-large-v1` embedding model, a state-of-the-art model that outperforms OpenAI `text-embedding-v3`, and I was interested in using it.

Fortunately Typesense makes this easy. I'm running it in docker, with a folder `data` for all related data. To set up Typesense to automatically create an embedding as it indexes using your custom model, first create the folder `data/models/mxbai-embed-large-v1`. Mixed Bread's model is available on [HuggingFace](https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1/tree/main/onnx), ONNX format is required.

Here are the steps I took to configure and use the model. I downloaded the fp16 version of the model, and renamed it to `model.onnx`, putting it in `data/models/mxbai-embed-large-v1/model.onnx`, I also downloaded the associated `vocab.txt` and put it in the same place. Lastly create a `config.json` with the following content:

```json
{
  "model_type": "bert",
  "vocab_file_name": "vocab.txt"
}
```

## Schema setup

Now, when creating the schema for your collection, you can configure additional fields that will be embeddings of text fields:

```python
schema = {
    "name": "sites",
    "fields": [
        {"name": "title", "type": "string"},
        {"name": "author", "type": "string", "optional": True},
        {"name": "url", "type": "string"},
        {"name": "text", "type": "string", "optional": True},
        {
            "name": "text_embedding",
            "type": "float[]",
            "embed": {
                "from": ["text"],
                "model_config": {
                    "model_name": "mxbai-embed-large-v1",
                },
            },
        },
    ],
}
create_response = client.collections.create(schema)
```

All the work will be done for you at index time. Performance can be slowed by this (I've seen docs take up to 4 seconds to index), but this can be helped by working in bulk or parallelizing requests. Querying is fast and accurate, which is exactly the trade-off I'd want to see.

Typesense is designed without any sort of UI, so I've been using [typesense-ui](https://github.com/philipeachille/typesense-ui) to browse records, pull random ids for testing etc and it has worked well so far. Overall I've been very impressed.
