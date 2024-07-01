---
title: "[隨筆] Postgres Dinstinct and Group By"
categories:
  - 隨筆
tags:
  - Postgres
toc: true
toc_label: Index
mermaid: true
---

Distinct 可以達成類似 Group by 效果


目前需求是   
table中 gantry,type,alert_type 會重複的, 想使用這三個欄位做group by, 找出最新的一筆資料   
但group by 做聚合的函數 , 只能找出特定欄位聚合後結果(e..g 基於gantry,type,alert event_start最新)   

```sql
SELECT
    ear.gantry,
    ear.type,
    ear.alert_type,
    max(ear.event_start)
FROM etc_alert_record AS ear
GROUP BY ear.gantry, ear.type, ear.alert_type
```
若我想看到 event_start最新的event_end就會沒辦法一起找出來  

<br/>


使用distinct on 可以做出類似 group by的效果   
這邊邏輯是先order by distinct on 的欄位, 這樣就會基於這些欄位做 排序, 因gantry,type,alert type 會相同, 會有類似group的效果  
order by最後一個則是 真正想排序的column  , 然後 Distinct on 可以基於 gantry,type,alert type 去重複, 會保留第一個  
這樣就可以做出基於 (ear.gantry, ear.type, ear.alert_type) group by 的效果, 並找出 event_start最新的 row  


```sql
SELECT DISTINCT ON (ear.gantry, ear.type, ear.alert_type)
    ear.gantry,
    ear.type,
    ear.alert_type,
    ear.event_start,
    ear.event_end
FROM etc_alert_record AS ear
ORDER BY ear.gantry, ear.type, ear.alert_type, ear.event_start DESC;
```



