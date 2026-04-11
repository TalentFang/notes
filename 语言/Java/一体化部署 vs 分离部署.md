
# 前后端一体化部署 vs 分离部署

## 一、一体化部署方案

### 1.1 什么是前后端一体化部署

将前端构建产物和后端应用打包在**同一个部署单元**中，通过同一个服务进程对外提供服务。

```
┌─────────────────────────────────────────┐
│           一体化部署架构                  │
├─────────────────────────────────────────┤
│  用户浏览器                               │
│      │                                   │
│      ▼                                   │
│  ┌─────────────────────────────────┐    │
│  │     Spring Boot 应用 (端口 8080) │    │
│  ├─────────────────────────────────┤    │
│  │  /api/*    → Controller 处理     │    │
│  │  /static/* → 前端静态资源        │    │
│  │  /*        → index.html (SPA)   │    │
│  └─────────────────────────────────┘    │
│      │                                   │
│      ▼                                   │
│  ┌─────────────────────────────────┐    │
│  │     MySQL / Redis / 其他         │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 1.2 实现方式

#### 方式一：将前端资源放入 Spring Boot 静态资源目录

```bash
# 项目结构
myapp/
├── src/
│   ├── main/
│   │   ├── java/           # 后端代码
│   │   └── resources/
│   │       ├── static/      # 前端静态资源放这里
│   │       │   ├── index.html
│   │       │   ├── css/
│   │       │   └── js/
│   │       └── application.yml
│   └── frontend/            # 前端源码（可选）
```

```yaml
# application.yml
spring:
  web:
    resources:
      static-locations: classpath:/static/
      add-mappings: true
```

#### 方式二：Maven/Gradle 插件自动构建

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <!-- 前端构建插件 -->
        <plugin>
            <groupId>com.github.eirslett</groupId>
            <artifactId>frontend-maven-plugin</artifactId>
            <version>1.12.1</version>
            <executions>
                <execution>
                    <id>install node and npm</id>
                    <phase>generate-resources</phase>
                    <goals>
                        <goal>install-node-and-npm</goal>
                    </goals>
                </execution>
                <execution>
                    <id>npm install</id>
                    <phase>generate-resources</phase>
                    <goals>
                        <goal>npm</goal>
                    </goals>
                </execution>
                <execution>
                    <id>npm build</id>
                    <phase>generate-resources</phase>
                    <goals>
                        <goal>npm</goal>
                    </goals>
                    <configuration>
                        <arguments>run build</arguments>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        
        <!-- 复制前端产物到静态资源目录 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy-frontend</id>
                    <phase>process-resources</phase>
                    <goals>
                        <goal>copy-resources</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/classes/static</outputDirectory>
                        <resources>
                            <resource>
                                <directory>${project.basedir}/frontend/dist</directory>
                            </resource>
                        </resources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### 方式三：后端配置 SPA 路由支持

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        // 将所有非 API 请求转发到 index.html
        registry.addViewController("/{path:^(?!api|static|actuator).*$}")
                .setViewName("forward:/index.html");
    }
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/")
                .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS));
    }
}
```

#### 方式四：使用 Spring Boot 的欢迎页面

```java
@Controller
public class IndexController {
    
    @GetMapping("/")
    public String index() {
        return "forward:/index.html";
    }
    
    // 或者直接返回视图名
    @GetMapping("/{path:[^\\.]*}")  // 匹配不含点的路径
    public String forward() {
        return "forward:/index.html";
    }
}
```

---

## 二、两种部署模式对比

### 2.1 架构对比表

| 维度         | 一体化部署        | 分离部署               |
| ---------- | ------------ | ------------------ |
| **部署单元**   | 单个 Jar/War 包 | 前端静态资源 + 后端服务      |
| **服务器数量**  | 1 台应用服务器     | 前端服务器 + 后端服务器      |
| **进程数**    | 1 个          | 2 个或更多（Nginx + 后端） |
| **域名**     | 同域           | 可同域或跨域             |
| **扩展方式**   | 垂直扩展（增加实例）   | 水平扩展（分别扩展）         |
| **CDN 支持** | 困难           | 原生支持               |

### 2.2 详细优劣势分析

#### 一体化部署优势

| 优势 | 说明 |
|------|------|
| **部署简单** | 一个 Jar 包，一条命令启动，适合小团队 |
| **运维成本低** | 无需管理 Nginx、无需配置代理和负载均衡 |
| **无跨域问题** | 前后端同域，无需 CORS 配置 |
| **事务一致性** | 可在同一事务中处理静态资源和业务逻辑 |
| **适合微服务内聚** | 每个微服务可独立打包前端，服务自治 |

#### 一体化部署劣势

| 劣势 | 说明 |
|------|------|
| **无法独立扩展** | 前端是纯静态，却要占用后端计算资源 |
| **无法用 CDN** | 静态资源无法享受 CDN 加速 |
| **构建耦合** | 前端改个文案也要重新构建后端 Jar |
| **无法独立发布** | 前后端发布绑定，无法做到前端独立上线 |
| **静态资源占用内存** | 每次请求都经过 Spring 处理，浪费 CPU |
| **不利于大团队协作** | 前后端代码在同一仓库，容易冲突 |

#### 分离部署优势

| 优势 | 说明 |
|------|------|
| **独立扩展** | 前端可独立扩展（CDN），后端按需扩容 |
| **CDN 加速** | 静态资源可享受 CDN 边缘节点加速 |
| **独立发布** | 前后端可独立上线，互不影响 |
| **技术栈解耦** | 前端可用任意框架，后端可用任意语言 |
| **性能更好** | Nginx 处理静态资源远快于 Java |
| **团队协作** | 前后端团队可独立开发、测试、部署 |

#### 分离部署劣势

| 劣势 | 说明 |
|------|------|
| **部署复杂** | 需要配置 Nginx、负载均衡、CORS 等 |
| **运维成本高** | 需要维护更多基础设施 |
| **跨域问题** | 需要处理 CORS 配置，增加复杂度 |
| **调试困难** | 本地开发需要配置代理 |
| **事务边界** | 无法在服务端统一处理页面和 API 的事务 |

---

## 三、实战对比：同一个应用两种部署方式

### 3.1 一体化部署示例

```bash
# 1. 构建
mvn clean package

# 2. 部署（单命令）
scp target/myapp.jar user@server:/opt/
ssh user@server "java -jar /opt/myapp.jar"

# 3. 访问
curl http://server:8080/          # 返回 HTML
curl http://server:8080/api/users # 返回 JSON
```

**Docker 部署**：
```dockerfile
FROM openjdk:11
COPY target/myapp.jar /app.jar
# 前端已打包在 jar 内
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 3.2 分离部署示例

```bash
# 前端部署
npm run build
scp -r dist/* user@nginx:/var/www/myapp/

# 后端部署
scp target/myapp.jar user@backend:/opt/
ssh user@backend "java -jar /opt/myapp.jar"

# Nginx 配置
sudo vim /etc/nginx/sites-available/myapp
sudo systemctl reload nginx

# 访问
curl http://server/           # Nginx 返回 HTML
curl http://server/api/users  # Nginx 代理到后端
```

---

## 四、混合部署方案（最佳实践）

### 4.1 开发环境：一体化
```yaml
# 开发阶段使用一体化，简化环境搭建
spring:
  profiles: dev
  web:
    resources:
      static-locations: file:./frontend/dist/
```

### 4.2 生产环境：分离
```nginx
# 生产环境使用 Nginx + CDN
upstream backend {
    server backend1:8080;
    server backend2:8080;
}

server {
    listen 443 ssl;
    
    # 静态资源走 CDN 或本地缓存
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # HTML 不缓存
    location / {
        alias /var/www/dist/;
        try_files $uri $uri/ /index.html;
        add_header Cache-Control "no-cache";
    }
    
    # API 代理到后端
    location /api/ {
        proxy_pass http://backend;
    }
}
```

### 4.3 渐进式迁移策略

```
阶段1: 一体化部署（启动阶段）
    ↓
阶段2: 分离部署（但保持同域，通过 Nginx 代理）
    ↓
阶段3: 完全分离（前端 CDN + 后端独立域名）
```

---

## 五、选择建议

### 5.1 选择一体化的场景

| 场景 | 原因 |
|------|------|
| **小型项目/原型验证** | 快速上线，运维简单 |
| **内部管理系统** | 用户量小，无 CDN 需求 |
| **全栈开发团队** | 开发人员同时负责前后端 |
| **服务内聚性要求高** | 每个微服务自带 UI（如运维平台） |
| **对部署复杂度敏感** | 缺乏专业运维人员 |

### 5.2 选择分离的场景

| 场景 | 原因 |
|------|------|
| **大型互联网应用** | 高并发、需要 CDN |
| **前后端团队分离** | 独立开发、独立部署 |
| **需要独立扩展** | 前端静态资源量级大 |
| **已有成熟 DevOps 体系** | 自动化部署能力完善 |
| **微服务架构** | 前端需要聚合多个后端服务 |

### 5.3 决策矩阵

```
                     分离复杂度
                         ↑
                         │
    推荐：一体化         │        推荐：分离
    （小团队、原型）      │        （大团队、高并发）
                         │
    推荐：混合           │        推荐：分离
    （过渡阶段）          │        （成熟 DevOps）
                         │
    └────────────────────┴────────────────────→ 流量规模
                     低                   高
```

---

## 六、总结

| 对比项 | 一体化部署 | 分离部署 |
|--------|-----------|---------|
| **部署难度** | ⭐ 简单 | ⭐⭐⭐ 复杂 |
| **运维成本** | ⭐ 低 | ⭐⭐⭐ 高 |
| **扩展性** | ⭐⭐ 一般 | ⭐⭐⭐⭐⭐ 优秀 |
| **性能（静态资源）** | ⭐⭐ 一般 | ⭐⭐⭐⭐⭐ 优秀（CDN） |
| **团队协作** | ⭐⭐ 耦合 | ⭐⭐⭐⭐⭐ 解耦 |
| **发布效率** | ⭐⭐ 绑定 | ⭐⭐⭐⭐⭐ 独立 |
| **适用场景** | 中小项目、内部系统 | 大型应用、高并发 |

**核心观点**：
- **没有绝对的好坏**，只有适不适合
- **初创项目**可以从一体化开始，快速验证
- **成长阶段**逐步过渡到分离部署
- **大型应用**必须分离，否则会成为瓶颈

现代开发实践中，**分离部署是主流**，因为它赋予了团队更大的灵活性和扩展能力。但一体化部署在某些场景下（如 B 端产品、内部工具）依然有其独特的价值。