# 交付物 0：本地环境搭建踩坑记录

## 改动文件

| 文件 | 改动 |
|------|------|
| `.env` | 创建，配置 PG/Redis/RustFS 凭证 |
| `application.yml` | `access-key` / `secret-key` 改为环境变量引用（无默认值） |

## 踩坑记录

### 1. RustFS S3 返回 500

**现象**：Spring Boot 启动时 `FileStorageService.ensureBucketExists()` 报 `Internal Privoxy Error (Status Code: 500)`。

**诊断链路**：
1. `docker exec interview-rustfs sh -c 'echo $RUSTFS_ACCESS_KEY'` → 确认容器凭证正确
2. `curl -s -o NUL -w "%{http_code}" http://localhost:9000` → 返回 500
3. 错误信息含 `Privoxy` 关键字 → 怀疑代理拦截
4. `echo $HTTP_PROXY` → 输出 `http://127.0.0.1:82`
5. 本机运行了本地代理，AWS S3 SDK 的 `localhost:9000` 请求被代理转发

**根因**：`HTTP_PROXY=http://127.0.0.1:82` 拦截了所有 HTTP 流量，包括 `localhost:9000`，代理无法转发 S3 协议请求。

**修复**：
```bash
export NO_PROXY="localhost,127.0.0.1,::1"
export no_proxy="localhost,127.0.0.1,::1"
```

### 2. Java 版本不匹配

**现象**：`./gradlew :app:compileJava` 报语法错误。

**诊断**：`java -version` → `OpenJDK 1.8.0_462`。项目要求 Java 21。

**根因**：系统默认 PATH 指向 `C:\Program Files\Amazon Corretto\jdk1.8.0_462`。

**修复**：Scoop 已安装 `temurin21-jdk`，使用绝对路径：
```bash
export JAVA_HOME="C:/Users/amazi/scoop/apps/temurin21-jdk/current"
"$JAVA_HOME/bin/java" -version  # 21.0.11 LTS
```

### 3. Gradle Daemon 不继承 JAVA_HOME

**现象**：设置 `JAVA_HOME` 后直接 `./gradlew` 仍用 Java 8。

**根因**：Gradle Wrapper 启动 JVM 时使用系统 JAVA_HOME，而不是当前 shell 的变量。

**修复**：使用完整命令链：
```bash
export JAVA_HOME="C:/Users/amazi/scoop/apps/temurin21-jdk/current"
export PATH="$JAVA_HOME/bin:$PATH"
./gradlew :app:bootRun --no-daemon  # --no-daemon 避免缓存旧 JVM
```

---

→ 详细环境搭建文档见 [my-learning/notes/01-env-setup.md](../../notes/01-env-setup.md)
