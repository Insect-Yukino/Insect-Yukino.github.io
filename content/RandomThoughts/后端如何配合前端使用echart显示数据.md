+++
date = '2025-11-28T21:41:21+08:00'
draft = false
title = '后端如何配合前端使用echart显示数据'
+++

```java
@RestController
@RequestMapping("/analysis")
public class AnalysisController {

    @GetMapping("/study-stat")
    public Map<String, Object> getStudyStat() {

        List<String> xAxis = List.of(
                "第一章",
                "第二章",
                "第三章",
                "第四章",
                "第五章"
        );

        List<Integer> data = List.of(
                120, 200, 150, 80, 70
        );

        Map<String, Object> result = new HashMap<>();
        result.put("xAxis", xAxis);
        result.put("data", data);

        return result;
    }
}
```

返回给前端的数据结构是这样的

```json
{
  "xAxis": ["第一章", "第二章", "第三章", "第四章", "第五章"],
  "data": [120, 200, 150, 80, 70]
}
```
