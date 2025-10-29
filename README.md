# 一、**基于 Word 模板占位符的动态文档生成技术**

> 💡 **作者**：古渡蓝按
>
> **个人微信公众号**：微信公众号（深入浅出谈java）
> 感觉本篇对你有帮助可以关注一下，会不定期更新知识和面试资料、技巧！！！

### 📝 简介

在企业业务系统中，合同、工单、报告等 Word 文档往往格式固定但内容动态。传统硬编码方式开发效率低、维护成本高。
本文介绍一种高效、灵活的解决方案：**通过预定义 Word 模板中的 `${KEY}` 占位符，结合后端数据自动填充生成最终文档**。该方法实现逻辑清晰、模板可由非技术人员维护，显著提升开发效率与系统可扩展性。以下是代码实现步骤和逻辑。

## 二、添加依赖:**Apache POI**

```
        <dependency>
            <groupId>org.apache.poigroupId>
            <artifactId>poi-ooxmlartifactId>
            <version>5.2.4version>
        dependency>

        
        <dependency>
            <groupId>org.projectlombokgroupId>
            <artifactId>lombokartifactId>
            <optional>trueoptional>
        dependency>
```

## 三、制作代占位符的word 模板

打开需要生产的数据模板，在对应位置填写占位符，类似下图：**占位符格式为：${XXXXX}**

📌 **注：占位符里面的必须和代码中的key 值一样**

制作完成后，放到 `src/main/resources/templates/` 目录下作为模板文件：

```
src/main/resources/templates/production_order_template.docx
```

**图片示例**

![image-20251029155045954](https://img2024.cnblogs.com/blog/2719585/202510/2719585-20251029165748315-1410650026.png)

## 四、编写核心逻辑

### **Controller 代码**

```
@Slf4j
@RestController
public class ProductionOrderController {



    @Resource
    private WordGeneratorService productionOrderService;

    @GetMapping("/api/generate-word")
    public void generateWord(@RequestParam Long id, HttpServletResponse response) throws IOException {

        ProductionOrder order = new ProductionOrder();

        byte[] docBytes = productionOrderService.generateProductionOrderDoc(order);

        // 设置正确的 Content-Type
        response.setContentType("application/vnd.openxmlformats-officedocument.wordprocessingml.document");
        response.setContentLength(docBytes.length);

        // ✅ 安全设置带中文的文件名（关键！）
        String filename = "生产任务单_" + id + ".docx";
        String encodedFilename = URLEncoder.encode(filename, StandardCharsets.UTF_8).replace("+", "%20");

        // 使用 filename* 语法（RFC 5987）：支持 UTF-8 文件名
        response.setHeader("Content-Disposition",
                "attachment; filename=\"" + encodedFilename + "\"; filename*=UTF-8''" + encodedFilename);

        // 写入响应体
        response.getOutputStream().write(docBytes);
        response.getOutputStream().flush();
    }

    @GetMapping("/api/generate-word2")
    public void generateWord2(@RequestParam String no, HttpServletResponse response) throws IOException {
        // 这里的ProductionOrder 可以换成自己对应的实体或者需要填写到数据库的对象
        // 正常逻辑是，这个order 是需要查后台数据，然后返回order对象，再在后续做模板和值 映射,类似下列代码，
        // 这一步最好放到实现类去写，这里只是为了方便
        //TODO:List getProductDataList = this.list(
        //        new LambdaQueryWrapper()
        //                .eq(ProductionOrder::getNo, no));
        ProductionOrder order = new ProductionOrder();

        // 改用模板生成
        byte[] docBytes = productionOrderService.generateFromTemplate(order);


        // 设置正确的 Content-Type
        response.setContentType("application/vnd.openxmlformats-officedocument.wordprocessingml.document");
        response.setContentLength(docBytes.length);

        // ✅ 安全设置带中文的文件名（关键！）
        String filename = "生产任务单_" + no + ".docx";
        String encodedFilename = URLEncoder.encode(filename, StandardCharsets.UTF_8).replace("+", "%20");

        // 使用 filename* 语法（RFC 5987）：支持 UTF-8 文件名
        response.setHeader("Content-Disposition",
                "attachment; filename=\"" + encodedFilename + "\"; filename*=UTF-8''" + encodedFilename);

        // 写入响应体
        response.getOutputStream().write(docBytes);
        response.getOutputStream().flush();
    }
}
```

### **Service层核心实现代码**

📌 **注：这里就省去了接口层（需要可以自己加），直接放置的核心方法**

```
@Service
public class WordGeneratorService {

    public byte[] generateProductionOrderDoc(ProductionOrder order) {
        try (XWPFDocument document = new XWPFDocument()) {
            // 标题
            XWPFParagraph titlePara = document.createParagraph();
            titlePara.setAlignment(ParagraphAlignment.CENTER);
            XWPFRun titleRun = titlePara.createRun();
            titleRun.setText("生产任务单申请表");
            titleRun.setFontSize(16);
            titleRun.setBold(true);

            // 创建表格（20列模拟原表宽度，实际按内容合并）
            XWPFTable table = document.createTable(5, 4);
            table.setWidth("100%");

            // 第一行：客户单位 & 订单号
            setCellText(table.getRow(0).getCell(0), "客户单位：");
            setCellText(table.getRow(0).getCell(1), order.getCustomer());
            setCellText(table.getRow(0).getCell(2), "订单号/合同编号：");
            setCellText(table.getRow(0).getCell(3), order.getOrderNo());

            // 第二行：产品名称 & 型号
            setCellText(table.getRow(1).getCell(0), "产品名称：");
            setCellText(table.getRow(1).getCell(1), order.getProductName());
            setCellText(table.getRow(1).getCell(2), "产品型号：");
            setCellText(table.getRow(1).getCell(3), order.getModel());

            // 第三行：规格（电压、电流、数量）
            setCellText(table.getRow(2).getCell(0), "规格");
            setCellText(table.getRow(2).getCell(1), "电压：" + order.getVoltage());
            setCellText(table.getRow(2).getCell(2), "电流：" + order.getCurrent());
            setCellText(table.getRow(2).getCell(3), "数量：" + order.getQuantity());

            // 第四行：生产周期
            setCellText(table.getRow(3).getCell(0), "生产周期");
            setCellText(table.getRow(3).getCell(1), "计划出货日期：" + order.getPlannedShipDate());
            setCellText(table.getRow(3).getCell(2), "销售项目人：");
            setCellText(table.getRow(3).getCell(3), order.getSalesPerson());

            // 第五行：备注或其他
            setCellText(table.getRow(4).getCell(0), "其他要求：");
            table.getRow(4).getCell(1).getParagraphs().get(0);

            // 合并单元格（可选，简化处理）
            // 实际复杂表格建议用模板或 Apache POI 高级合并

            ByteArrayOutputStream out = new ByteArrayOutputStream();
            document.write(out);
            return out.toByteArray();
        } catch (Exception e) {
            throw new RuntimeException("生成 Word 失败", e);
        }
    }

    private void setCellText(XWPFTableCell cell, String text) {
        cell.setText(text);
        // 可选：设置字体
        for (XWPFParagraph p : cell.getParagraphs()) {
            for (XWPFRun r : p.getRuns()) {
                r.setFontFamily("宋体");
                r.setFontSize(10);
            }
        }
    }


    //方式二

    private static final String TEMPLATE_PATH = "templates/production_order_template.docx";

    public byte[] generateFromTemplate(ProductionOrder order) {
        try {
            // 1. 加载模板
            ClassPathResource resource = new ClassPathResource(TEMPLATE_PATH);
            try (InputStream is = resource.getInputStream();
                 ByteArrayOutputStream out = new ByteArrayOutputStream()) {

                XWPFDocument document = new XWPFDocument(is);

                // 2. 构建数据映射
                Map data = new HashMap<>();
                data.put("customer", safeStr(order.getCustomer()));
                data.put("orderNo", safeStr(order.getOrderNo()));
                data.put("workOrderNo", safeStr(order.getWorkOrderNo()));
                data.put("productName", safeStr(order.getProductName()));
                data.put("model", safeStr(order.getModel()));
                data.put("voltage", safeStr(order.getVoltage()));
                data.put("current", safeStr(order.getCurrent()));
                data.put("quantity", safeStr(order.getQuantity() != null ? order.getQuantity().toString() : ""));
                data.put("plannedShipDate", safeStr(order.getPlannedShipDate()));
                data.put("salesPerson", safeStr(order.getSalesPerson()));
                
                
                //如果你希望某些字段只显示“√”表示选中，可以在 Java 中这样处理：
                data.put("hasEmbeddedSeal", order.isEmbeddedSeal() ? "√" : "");


                

                // 3. 替换所有段落中的占位符
                replaceInParagraphs(document.getParagraphs(), data);

                // 4. 替换表格中的占位符
                for (XWPFTable table : document.getTables()) {
                    for (XWPFTableRow row : table.getRows()) {
                        for (XWPFTableCell cell : row.getTableCells()) {
                            replaceInParagraphs(cell.getParagraphs(), data);
                        }
                    }
                }

                // 5. 输出为字节数组
                document.write(out);
                return out.toByteArray();
            }
        } catch (Exception e) {
            throw new RuntimeException("生成 Word 文档失败", e);
        }
    }

    /**
     * 替换段落中的占位符
     */
    private void replaceInParagraphs(List paragraphs, Map data) {
        for (XWPFParagraph para : paragraphs) {
            for (XWPFRun run : para.getRuns()) {
                if (run != null && run.getText(0) != null) {
                    String text = run.getText(0);
                    String replaced = replacePlaceholders(text, data);
                    if (!text.equals(replaced)) {
                        run.setText(replaced, 0);
                    }
                }
            }
        }
    }

    /**
     * 使用正则替换 ${key} 为 value
     */
    private String replacePlaceholders(String text, Map data) {
        Pattern pattern = Pattern.compile("\\$\\{([^}]+)\\}");
        Matcher matcher = pattern.matcher(text);
        StringBuffer sb = new StringBuffer();
        while (matcher.find()) {
            String key = matcher.group(1);
            String replacement = data.getOrDefault(key, matcher.group(0)); // 未找到则保留原样
            matcher.appendReplacement(sb, replacement == null ? "" : Matcher.quoteReplacement(replacement));
        }
        matcher.appendTail(sb);
        return sb.toString();
    }

    private String safeStr(String str) {
        return str == null ? "" : str;
    }
}
```

## 五、注意事项

### ⚠️ **1、占位符被拆分问题（未能正确显示数值）**

Word 会因格式变化将 `${NO}` 拆成多个 `Run`（如 `${N` + `O}`），导致无法匹配。这里不要用文本框或艺术字等

Apache POI 在读取 Word 文档时，会将文本按格式（字体、颜色、加粗等）拆分成多个 **`XWPFRun`** 对象。

例如下面图片，编号未能正确显示

![image-20251029163259173](https://img2024.cnblogs.com/blog/2719585/202510/2719585-20251029165832778-1129694254.png)

**❌ 问题场景：**

如果在 Word 中输入 `${NO}` 时：

* 中间不小心按了方向键、空格、Backspace
* 或对部分字符设置了格式（比如只加粗了 `N`）
* 或从其他地方复制粘贴过来

那么 Word 内部可能存储为：

```
Run1: "${N"
Run2: "O}"
```

而替换逻辑是 **逐 `Run` 处理**：

```
for (XWPFRun run : para.getRuns()) {
    String text = run.getText(0); // 只拿到 "${N" 或 "O}"
    // 无法匹配完整 "${NO}"
}
```

→ **结果：`${NO}` 没有被识别，也就不会被替换！**

而其他占位符（如 `${SJBBH}`）可能是一次性输入的，所以在一个 `Run` 里，能正常替换。

**解决方案**：

* **在模板中一次性输入完整占位符，避免中途格式调整。（不要中途按方向键、不要设置局部格式）**

  > 💡 技巧：可以先输入 `ABC`，确认它在一个 Run 里（比如全选后统一加粗），再替换成 `${NO}`。
* **或使用更高级的跨 `Run` 合并替换算法（实现复杂）。**

  当前逻辑只处理单个 Run，无法处理被拆分的占位符。可以改用更健壮的方案：

  方案 A：合并段落所有文本，整体替换（简单但会丢失格式，不推荐，会破坏原有样式）

  方案 B：使用递归或缓冲区拼接 Run（复杂），但对大多数项目来说，**方法 1（规范模板输入）是最高效、最可靠的**。

🔧 **调试技巧**：如果替换失败，可临时打印 `run.getText(0)` 查看实际文本分段。

* **次要可能原因排查**

  + ### ✅ 1. 检查 Java 实体类字段是否正确
  + ### 2. 检查 Word 模板中是否真的是 `${NO}`（大小写敏感）
  + ### 检查是否在表格 or 段落中？

---

### ⚠️**2、使用方式一，返回的是zip文件而不是word 文件**

核心原因：

**`.docx` 文件本质上就是一个 ZIP 压缩包**！

* Microsoft Office 2007 及以后的 `.docx`、`.xlsx`、`.pptx` 文件都采用 **Open XML 格式**。
* 这种格式实际上是将 XML、图片、样式等文件打包成一个 **ZIP 压缩包**，只是扩展名改成了 `.docx`。
* 当用代码生成.docx但没有正确设置 HTTP 响应头（Content-Type 和 Content-Disposition）
  + 浏览器无法识别这是 Word 文档
  + 会根据文件内容的“真实类型”（ZIP）来处理
  + 于是**自动下载为 `.zip` 文件**，或提示“文件损坏”

**解决方案**：

* 设置正确的响应头

```
HttpHeaders headers = new HttpHeaders();

// 1. 设置 Content-Type（MIME 类型）
headers.setContentType(MediaType.parseMediaType("application/vnd.openxmlformats-officedocument.wordprocessingml.document"));

// 2. 设置 Content-Disposition（告诉浏览器这是附件，且文件名是 .docx）
headers.setContentDispositionFormData("attachment", "生产任务单.docx");

return new ResponseEntity<>(docBytes, headers, HttpStatus.OK);
```

**❌ 常见错误写法（会导致 ZIP 下载）：**

```
// 错误1：Content-Type 写成 application/zip 或 application/octet-stream
headers.setContentType(MediaType.APPLICATION_OCTET_STREAM); // ❌

// 错误2：文件名没有 .docx 后缀
headers.setContentDispositionFormData("attachment", "report"); // ❌ 下载为 report.zip

// 错误3：文件名包含非法字符（如 / \ : * ? " < > |）
headers.setContentDispositionFormData("attachment", "生产/任务单.docx"); // ❌ 可能被截断或变 ZIP
```

🔧 额外检查点：

1. **确认生成的字节数组确实是合法 `.docx`**

   * 将 `docBytes` 保存到本地文件：`Files.write(Paths.get("test.docx"), docBytes);`
   * 用 Word 能正常打开吗？如果打不开 → 说明生成逻辑有误（不是 ZIP 问题，是文件损坏）
2. **不要用 `application/zip` 或 `application/octet-stream`**
   即使内容是 ZIP 结构，也必须声明为 Word 的 MIME 类型！

---

### ⚠️**3、使用浏览器直接请求报错**

报错示例：

```
java.lang.IllegalArgumentException: The Unicode character [生] at code point [29,983] cannot be encoded as it is outside the permitted range of 0 to 255
```

**根本原因**：
在设置 HTTP 响应头（特别是 `Content-Disposition` 文件名）时，**直接使用了包含中文字符（如“生产任务单.docx”）的字符串**，而 Tomcat 在处理 HTTP 响应头时，默认使用 **ISO-8859-1 编码**（只支持 0–255 的字节范围），无法表示中文字符（Unicode 超出 255），于是抛出异常。

✅ 正确解决方案：对文件名进行 **RFC 5987 / RFC 2231 兼容的编码**

HTTP 协议规定：**响应头中的非 ASCII 字符必须进行编码**。推荐使用 **`filename\*` 语法（带编码声明）**。

```
// 设置正确的 Content-Type
    response.setContentType("application/vnd.openxmlformats-officedocument.wordprocessingml.document");
    response.setContentLength(docBytes.length);

    // ✅ 安全设置带中文的文件名（关键！）
    String filename = "生产任务单_" + id + ".docx";
    String encodedFilename = URLEncoder.encode(filename, StandardCharsets.UTF_8).replace("+", "%20");

    // 使用 filename* 语法（RFC 5987）：支持 UTF-8 文件名
    response.setHeader("Content-Disposition",
        "attachment; filename=\"" + encodedFilename + "\"; filename*=UTF-8''" + encodedFilename);

    // 写入响应体
    response.getOutputStream().write(docBytes);
    response.getOutputStream().flush();
```

## 六、接口验证

可以访问接口：[http://127.0.0.1:8199/api/generate-word2?no=27202SCRW250006](https://github.com)

这样你的浏览器就会弹出下载页面，并且获取一个填充数据的word 文档

![image-20251029164051702](https://img2024.cnblogs.com/blog/2719585/202510/2719585-20251029165906142-91108606.png)

本博客参考[豆荚加速器官网](https://baitenghuo.com)。转载请注明出处！
