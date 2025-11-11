            if "score" in filter_:
                score = result.value["score"]
                for op, value in filter_["score"].items():
                    if op == "$gt":
                        assert score > value
                    elif op == "$gte":
                        assert score >= value
                    elif op == "$lt":
                        assert score < value
                    elif op == "$lte":
                        assert score <= value

            if "index" in filter_:
                index = result.value["index"]
                for op, value in filter_["index"].items():
                    if op == "$gt":
                        assert index > value
                    elif op == "$gte":
                        assert index >= value
                    elif op == "$lt":
                        assert index < value
                    elif op == "$lte":
                        assert index <= value


def test_vector_search_pagination(fake_embeddings: CharacterEmbeddings) -> None:
    """Test pagination with vector search."""
    store = InMemoryStore(
        index={"dims": fake_embeddings.dims, "embed": fake_embeddings}
    )
    for i in range(5):
        store.put(("test",), f"doc{i}", {"text": f"test document number {i}"})

    results_page1 = store.search(("test",), query="test", limit=2)
    results_page2 = store.search(("test",), query="test", limit=2, offset=2)

    assert len(results_page1) == 2
    assert len(results_page2) == 2
    assert results_page1[0].key != results_page2[0].key

    all_results = store.search(("test",), query="test", limit=10)
    assert len(all_results) == 5


async def test_async_vector_search_pagination(
    fake_embeddings: CharacterEmbeddings,
) -> None:
    """Test pagination with vector search using async methods."""
    store = InMemoryStore(
        index={"dims": fake_embeddings.dims, "embed": fake_embeddings}
    )
    for i in range(5):
        await store.aput(("test",), f"doc{i}", {"text": f"test document number {i}"})

    results_page1 = await store.asearch(("test",), query="test", limit=2)
    results_page2 = await store.asearch(("test",), query="test", limit=2, offset=2)

    assert len(results_page1) == 2
    assert len(results_page2) == 2
    assert results_page1[0].key != results_page2[0].key

    all_results = await store.asearch(("test",), query="test", limit=10)
    assert len(all_results) == 5


async def test_embed_with_path(fake_embeddings: CharacterEmbeddings) -> None:
    # Test store-level field configuration
    store = InMemoryStore(
        index={
            "dims": fake_embeddings.dims,
            "embed": fake_embeddings,
            # Key 2 isn't included. Don't index it.
            "fields": ["key0", "key1", "key3"],
        }
    )
    # This will have 2 vectors representing it
    doc1 = {
        # Omit key0 - check it doesn't raise an error
        "key1": "xxx",
        "key2": "yyy",
        "key3": "zzz",
    }
    # This will have 3 vectors representing it
    doc2 = {
        "key0": "uuu",
        "key1": "vvv",
        "key2": "www",
        "key3": "xxx",
    }
    await store.aput(("test",), "doc1", doc1)
    await store.aput(("test",), "doc2", doc2)

    # doc2.key3 and doc1.key1 both would have the highest score
    results = await store.asearch(("test",), query="xxx")
    assert len(results) == 2
    assert results[0].key != results[1].key
    ascore = results[0].score
    bscore = results[1].score
    assert ascore is not None and bscore is not None
    assert ascore == pytest.approx(bscore, abs=1e-5)

    results = await store.asearch(("test",), query="uuu")
    assert len(results) == 2
    assert results[0].key != results[1].key
    assert results[0].key == "doc2"
    assert results[0].score is not None and results[0].score > results[1].score
    assert ascore == pytest.approx(results[0].score, abs=1e-5)

    # Un-indexed - will have low results for both. Not zero (because we're projecting)
    # but less than the above.
    results = await store.asearch(("test",), query="www")
    assert len(results) == 2
    assert results[0].score < ascore
    assert results[1].score < ascore

    # Test operation-level field configuration
    store_no_defaults = InMemoryStore(
        index={
            "dims": fake_embeddings.dims,
            "embed": fake_embeddings,
            "fields": ["key17"],
        }
    )

    doc3 = {
        "key0": "aaa",
        "key1": "bbb",
        "key2": "ccc",
        "key3": "ddd",
    }
    doc4 = {
        "key0": "eee",
        "key1": "bbb",  # Same as doc3.key1
        "key2": "fff",
        "key3": "ggg",
    }

    await store_no_defaults.aput(("test",), "doc3", doc3, index=["key0", "key1"])
    await store_no_defaults.aput(("test",), "doc4", doc4, index=["key1", "key3"])

    results = await store_no_defaults.asearch(("test",), query="aaa")
    assert len(results) == 2
    assert results[0].key == "doc3"
    assert results[0].score is not None and results[0].score > results[1].score

    results = await store_no_defaults.asearch(("test",), query="ggg")
    assert len(results) == 2
    assert results[0].key == "doc4"
    assert results[0].score is not None and results[0].score > results[1].score

    results = await store_no_defaults.asearch(("test",), query="bbb")
    assert len(results) == 2
    assert results[0].key != results[1].key
    assert results[0].score == results[1].score

    results = await store_no_defaults.asearch(("test",), query="ccc")
    assert len(results) == 2
    assert all(r.score < ascore for r in results)

    doc5 = {
        "key0": "hhh",
        "key1": "iii",
    }
    await store_no_defaults.aput(("test",), "doc5", doc5, index=False)

    results = await store_no_defaults.asearch(("test",), query="hhh")
    assert len(results) == 3
    doc5_result = next(r for r in results if r.key == "doc5")
    assert doc5_result.score is None


def test_non_ascii(fake_embeddings: CharacterEmbeddings) -> None:
    """Test support for non-ascii characters"""
    store = InMemoryStore(
        index={"dims": fake_embeddings.dims, "embed": fake_embeddings}
    )
    store.put(("user_123", "memories"), "1", {"text": "这是中文"})  # Chinese
    store.put(("user_123", "memories"), "2", {"text": "これは日本語です"})  # Japanese
    store.put(("user_123", "memories"), "3", {"text": "이건 한국어야"})  # Korean
    store.put(("user_123", "memories"), "4", {"text": "Это русский"})  # Russian
    store.put(("user_123", "memories"), "5", {"text": "यह रूसी है"})  # Hindi

    result1 = store.search(("user_123", "memories"), query="这是中文")
    result2 = store.search(("user_123", "memories"), query="これは日本語です")
    result3 = store.search(("user_123", "memories"), query="이건 한국어야")
    result4 = store.search(("user_123", "memories"), query="Это русский")
    result5 = store.search(("user_123", "memories"), query="यह रूसी है")

    assert result1[0].key == "1"
    assert result2[0].key == "2"
    assert result3[0].key == "3"
    assert result4[0].key == "4"
    assert result5[0].key == "5"

```
---
