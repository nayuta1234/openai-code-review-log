## 代码评审：`ApiTest.java`

### 一、变更概述
本次变更将 `test_http()` 测试方法整体注释掉（`@Test` 与 `}` 均被包含在块注释中）。从 diff 内仅能判断这是临时禁用测试，但由于该方法内含敏感信息与真实网络调用，需要重点提示。

---

### 二、关键问题（必须处理）

#### 1. ⚠️ 硬编码密钥已进入 Git 历史
```java
String apiKeySecret = "c78fbacd3e10118ad5649d7a54a3a163.UunYDBxpzeClvSKZ";
```
即使现在被注释掉，该密钥**已经提交到版本库**，注释并不等于删除。任何能拉取仓库/历史记录的人都能拿到。攻击者可借此调用 API 产生费用或泄露数据。

**处置建议：**
- **立即在服务商侧吊销/轮换该 `apiKeySecret`**（这是最重要的一步，代码层面修复都无法挽回已泄露的密钥）。
- 后续密钥通过环境变量或配置中心注入：
  ```java
  String apiKeySecret = System.getenv("ZHIPU_API_KEY");
  ```
- 或者使用 `application-test.yml` + `@Value`，并确保该文件被 `.gitignore`。
- 如果仓库是公开或共享的，考虑使用 `git filter-repo` / BFG 清理历史（注意：清理历史需团队协作，且已经泄露过的密钥无论如何都要吊销）。

#### 2. 应删除死代码，而不是注释
注释掉整个方法留下了几十行不可执行代码，属于典型的"代码坟墓"。问题在于：
- 编译器不检查，渐渐与真实逻辑脱节；
- 阅读者无法确定它是"暂时禁用"还是"永久废弃"；
- 干扰 IDE 搜索与静态检查。

**推荐做法（按优先级）：**
1. **直接删除**——如果不再需要，Git 历史本身就是最好的备份。
2. **需要保留但禁用**，用 JUnit5 的 `@Disabled`，语义清晰：
   ```java
   @Test
   @Disabled("依赖真实 API，需要 ZHIPU_API_KEY，CI 环境不执行")
   public void testHttp() throws IOException { ... }
   ```
3. 若只是"手动联调工具"，应移到独立的 `manual` 包或 `main` 目录下的调试工具类，不要混在单元测试里。

#### 3. 使用严格的块注释语法
当前是：
```java
   /* @Test
    ...
    }*/
```
`@Test` 被吞进注释块、`}`紧贴 `*/`，不符合 Java 代码规范。

正确写法（若坚持注释）：
```java
    /*
    @Test
    public void testHttp() throws IOException {
        ...
    }
    */
```
或者每个成员单独 `//` 注释。但如上所述，**不推荐注释，推荐删除或 `@Disabled`**。

---

### 三、其他改进建议

| 项 | 现状 | 建议 |
|---|---|---|
| 方法命名 | `test_http` | 遵循驼峰：`testHttp()` |
| 测试性质 | 真实调用外网 HTTP + LLM 接口 | 单测不应依赖外网，建议用 `MockWebServer`/`WireMock` 返回固定响应 |
| 断言 | 仅 `System.out.println`，无任何断言 | 若保留测试，至少 `assertNotNull(response.getChoices())` 等，否则不是测试而是 demo |
| 空类风险 | 注释后 `ApiTest` 可能除 `test_token` 外无其他内容 | 若方法被删除后类为空，考虑删除整个类；`test_token()` 中若也含硬编码密钥，需一并处理 |

> 提示：`test_token()` 方法中大概率也存在硬编码的 `apiKeySecret`，建议一并排查整个仓库中是否还有其他硬编码凭证（可用 `git grep -nE "[0-9a-f]{32}\."` 之类粗略筛查）。

---

### 四、结论

| 级别 | 问题 |
|---|---|
| 🔴 阻塞 | 硬编码 API 密钥已泄露 → **立即吊销密钥**，并改为环境变量注入 |
| 🟠 高 | 用注释代替删除/禁用，产生死代码 → 改用 `@Disabled` 或直接删除 |
| 🟡 中 | 块注释语法不规范；测试无断言、依赖外网 |
| 🟢 低 | 方法命名风格 |

**最小改动版建议：**

```java
@Test
@Disabled("依赖真实 LLM API，手动联调时临时开启；密钥通过环境变量注入")
public void testHttp() throws IOException {
    String apiKeySecret = System.getenv("ZHIPU_API_KEY");
    assertNotNull(apiKeySecret, "缺少 ZHIPU_API_KEY 环境变量");
    String token = BearerTokenUtils.getToken(apiKeySecret);
    ...
    assertFalse(response.getChoices().isEmpty());
}
```

如果短期不再需要该联调能力，最干净的做法是**整段删除**并吊销已暴露的密钥。