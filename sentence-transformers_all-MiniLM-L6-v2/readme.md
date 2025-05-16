# OpenSearch 語義搜索實施SOP

本文檔提供了在OpenSearch中實現語義搜索的完整步驟流程，包括安裝、模型部署和使用。

## 目錄

- [設置語義搜索索引](#設置語義搜索索引)
- [索引文檔並生成嵌入](#索引文檔並生成嵌入)
- [執行語義搜索查詢](#執行語義搜索查詢)
- [OpenSearch Dashboards UI操作](#opensearch-dashboards-ui操作)
- [命令行操作](#命令行操作)
- [故障排除](#故障排除)


## 索引文檔並生成嵌入

在索引文檔之前，需要為每個文檔生成嵌入。以下是通過API正確索引文檔的流程：

### 通過命令行

#### 文檔1

```bash
# 生成嵌入
curl -XPOST https://localhost:9200/_plugins/_ml/_predict/text_embedding/YOUR_MODEL_ID -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d '{
  "text_docs": ["OpenSearch提供強大的搜索功能"],
  "return_number": true
}' --insecure > doc1_response.json

# 使用jq創建索引文檔
cat doc1_response.json | jq '{
  "content": "OpenSearch提供強大的搜索功能",
  "embedding": .inference_results[0].output[2].data
}' > doc1.json

# 索引文檔
curl -XPOST https://localhost:9200/semantic-search/_doc -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d @doc1.json --insecure
```

#### 文檔2

```bash
# 生成嵌入
curl -XPOST https://localhost:9200/_plugins/_ml/_predict/text_embedding/YOUR_MODEL_ID -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d '{
  "text_docs": ["機器學習可以改進搜索結果"],
  "return_number": true
}' --insecure > doc2_response.json

# 使用jq創建索引文檔
cat doc2_response.json | jq '{
  "content": "機器學習可以改進搜索結果",
  "embedding": .inference_results[0].output[2].data
}' > doc2.json

# 索引文檔
curl -XPOST https://localhost:9200/semantic-search/_doc -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d @doc2.json --insecure
```

#### 文檔3

```bash
# 生成嵌入
curl -XPOST https://localhost:9200/_plugins/_ml/_predict/text_embedding/YOUR_MODEL_ID -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d '{
  "text_docs": ["語義搜索幫助用戶找到相關內容"],
  "return_number": true
}' --insecure > doc3_response.json

# 使用jq創建索引文檔
cat doc3_response.json | jq '{
  "content": "語義搜索幫助用戶找到相關內容",
  "embedding": .inference_results[0].output[2].data
}' > doc3.json

# 索引文檔
curl -XPOST https://localhost:9200/semantic-search/_doc -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d @doc3.json --insecure
```

### 驗證索引的文檔

```bash
curl -XGET https://localhost:9200/semantic-search/_search -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d '{
  "query": {
    "match_all": {}
  }
}' --insecure
```

確認所有文檔都已被正確索引並包含嵌入向量。

## 執行語義搜索查詢

### 通過命令行進行語義搜索

```bash
# 生成查詢嵌入
curl -XPOST https://localhost:9200/_plugins/_ml/_predict/text_embedding/YOUR_MODEL_ID -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d '{
  "text_docs": ["如何使用AI改進搜索"],
  "return_number": true
}' --insecure > query_response.json

# 創建KNN查詢
cat query_response.json | jq '{
  "query": {
    "knn": {
      "embedding": {
        "vector": .inference_results[0].output[2].data,
        "k": 2
      }
    }
  }
}' > knn_query.json

# 執行搜索
curl -XGET https://localhost:9200/semantic-search/_search -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d @knn_query.json --insecure
```

## OpenSearch Dashboards UI操作

OpenSearch Dashboards提供了更直觀的界面來執行上述操作。以下是使用Dashboards的步驟：

### 1. 訪問Dashboards

在瀏覽器中訪問 http://localhost:5601 並使用管理員憑據登錄。

### 2. 創建索引

在左側導航欄中，選擇"Dev Tools"，然後執行：

```
PUT semantic-search
{
  "mappings": {
    "properties": {
      "content": { "type": "text" },
      "embedding": {
        "type": "knn_vector",
        "dimension": 384
      }
    }
  },
  "settings": {
    "index": {
      "knn": true,
      "knn.space_type": "cosinesimil"
    }
  }
}
```

### 3. 生成嵌入並索引文檔

對每個文檔執行以下操作：

```
# 生成嵌入
POST _plugins/_ml/_predict/text_embedding/YOUR_MODEL_ID
{
  "text_docs": ["OpenSearch提供強大的搜索功能"],
  "return_number": true
}

# 從響應中複製sentence_embedding的data數組

# 索引文檔
POST semantic-search/_doc
{
  "content": "OpenSearch提供強大的搜索功能",
  "embedding": [貼上從上一步複製的向量]
}
```

重複此過程以添加更多文檔。

### 4. 執行語義搜索

```
# 生成查詢嵌入
POST _plugins/_ml/_predict/text_embedding/YOUR_MODEL_ID
{
  "text_docs": ["如何使用AI改進搜索"],
  "return_number": true
}

# 使用生成的向量執行KNN搜索
GET semantic-search/_search
{
  "query": {
    "knn": {
      "embedding": {
        "vector": [貼上從上一步複製的向量],
        "k": 2
      }
    }
  }
}
```

### 5. 與傳統搜索比較

```
# 關鍵詞搜索
GET semantic-search/_search
{
  "query": {
    "match": {
      "content": "AI"
    }
  }
}
```

## 命令行操作

### API響應結構解析

了解OpenSearch ML模型的API響應結構非常重要。以下是關鍵JSON路徑：

- 嵌入向量: `.inference_results[0].output[2].data`
  - 其中 `output[2]` 是名為 `sentence_embedding` 的輸出
  - 此向量的維度為384

### 使用jq提取數據

jq是處理JSON數據的強大工具。以下是一些有用的命令：

```bash
# 顯示API響應的頂級結構
cat response.json | jq 'keys'

# 列出所有輸出的名稱
cat response.json | jq '.inference_results[0].output[].name'

# 檢查向量維度
cat response.json | jq '.inference_results[0].output[2].data | length'
```

## 故障排除

### 常見問題

1. **"Failed to find model" 錯誤**
   - 確保使用正確的模型ID
   - 檢查模型是否成功部署

2. **"vector doesn't support values of type: VALUE_NULL" 錯誤**
   - JSON路徑不正確，導致嘗試使用null值作為向量
   - 確保正確提取嵌入向量 (`.inference_results[0].output[2].data`)

3. **KNN搜索返回空結果**
   - 檢查文檔是否成功索引並包含嵌入
   - 確認查詢向量維度正確 (384)
   - 嘗試增加k值或降低相似度閾值

4. **嵌入生成失敗**
   - 檢查模型是否正確部署
   - 確認文本輸入格式正確

### 驗證步驟

遇到問題時，可以使用以下步驟進行驗證：

1. 檢查模型狀態：
   ```bash
   curl -XGET https://localhost:9200/_plugins/_ml/models/YOUR_MODEL_ID -u 'admin:YourStrongPassword' --insecure
   ```

2. 檢查索引映射：
   ```bash
   curl -XGET https://localhost:9200/semantic-search/_mapping -u 'admin:YourStrongPassword' --insecure
   ```

3. 檢查索引文檔：
   ```bash
   curl -XGET https://localhost:9200/semantic-search/_search -u 'admin:YourStrongPassword' -H 'Content-Type: application/json' -d '{
     "query": {
       "match_all": {}
     },
     "_source": ["content", "embedding"]
   }' --insecure
   ```

---

本SOP基於OpenSearch 2.19.1版本，使用all-MiniLM-L6-v2作為嵌入模型。不同版本的OpenSearch或不同的模型可能需要調整部分步驟。
