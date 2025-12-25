# 目的
本文件用於記錄 ESG 專案中資料庫 Schema 的設計演進過程，
包含欄位調整原因、結構變更背景與設計決策依據，
以利團隊溝通、後續維護與研究說明。


## 初版設計（會議前草案）

### Table 1：公司基本資料
- 公司
- 產業別
- 股票代號
- 年份
- 風險總分
- URL

### Table 2：ESG 報告分析
- esg_category
- page_number
- report_claim
- risk_score

### Table 3：外部新聞分析
- esg_category
- disclosure_claim
- external_evidence
- consistency_status
- risk_score

-----------------------------------------------------------
## Schema 調整動機

經小組討論後，發現初版設計存在以下問題：

1. 無法有效支援「同一公司、同一年、多筆分析結果」的情境
2. ESG 報告聲稱（claim）與外部新聞證據缺乏明確關聯鍵
3. 無法追溯分析結果對應的實際報告來源（頁碼、URL）
4. 不利於後續 Dashboard 查詢與風險排序

## 調整後 Schema（定稿版）

### dim_company（公司維度表）
- company_id (PK)
- stock_code（股票代號）
- company_name
- industry_category

### fact_report_summary（每公司每年一份報告）
- report_id (PK)
- company_id (FK)
- year
- total_risk_score
- report_url

### company_report_detail（ESG 報告內部分析）
- detail_id (PK)
- report_id (FK)
- esg_category (E / S / G)
- sasb_topic
- page_number
- report_claim
- greenwashing_factor
- internal_risk_score

### news_report_validation（外部新聞驗證）
- news_id (PK)
- report_id (FK)
- esg_category
- disclosure_claim
- external_evidence
- consistency_status
- external_risk_score
- source_url



## 關鍵設計決策說明

### 1. 使用 report_id 作為核心關聯鍵
- 原因：同一公司在不同年度會有不同 ESG 報告
- 效益：可同時支援跨年度比較與單一年度深入分析

### 2. 區分「內部揭露」與「外部驗證」為兩張表
- 內部揭露：ESG 報告書中的企業聲稱
- 外部驗證：新聞或官方資料對聲稱的一致性比對
- 效益：降低 LLM 幻覺風險、提高可追溯性

### 3. 風險分數統一採用 0–100 制
- 便於使用者理解
- 便於 Dashboard 視覺化與排序



## 未來擴充方向（尚未實作）

- 將 greenwashing_factor 正規化為獨立字典表
- 新增模型版本欄位以比較不同 LLM 分析結果
- 引入人工複核標記（human_verified）
- 支援多來源新聞比對（新聞、NGO、政府資料）