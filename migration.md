
## Train-Ticket 微服务升级指南

### 版本升级概览

| 组件 | 旧版本 | 新版本 |
|------|--------|--------|
| Spring Boot | 2.3.12.RELEASE | 3.2.0 |
| Spring Cloud | Hoxton.SR12 | 2023.0.0 |
| Spring Cloud Alibaba | 2.2.7.RELEASE | 2023.0.1.0 |
| Java | 8 | 17 |
| Swagger | Springfox 2.4.0 | SpringDoc OpenAPI 2.3.0 |
| JWT | jjwt 0.8.0 | jjwt 0.12.3 |
| Fastjson | 1.2.31 | Fastjson2 2.0.43 |

---

### 1. 子模块 pom.xml 更新

```xml
<!-- 旧版本 -->
<properties>
    <java.version>1.8</java.version>
</properties>

<!-- 新版本 -->
<properties>
    <java.version>17</java.version>
</properties>
```

---

### 2. Application 类更新

**移除：**
```java
import springfox.documentation.swagger2.annotations.EnableSwagger2;
@EnableSwagger2
```

**替换为：**
```java
import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Info;

@OpenAPIDefinition(info = @Info(
    title = "服务名称 API",
    version = "1.0",
    description = "服务描述"
))
```

---

### 3. SecurityConfig 更新

**旧版本（Spring Security 5）：**
```java
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import static org.springframework.web.cors.CorsConfiguration.ALL;

@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurerAdapter() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**")
                        .allowedOrigins(ALL)
                        .allowedMethods(ALL)
                        .allowedHeaders(ALL)
                        .allowCredentials(false)
                        .maxAge(3600);
            }
        };
    }

    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.httpBasic().disable()
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                .antMatchers("/api/v1/xxxservice/**").hasAnyRole("ADMIN", "USER")
                .antMatchers("/swagger-ui.html", "/webjars/**", "/images/**",
                        "/configuration/**", "/swagger-resources/**", "/v2/**").permitAll()
                .anyRequest().authenticated()
                .and()
                .addFilterBefore(new JWTFilter(), UsernamePasswordAuthenticationFilter.class);

        httpSecurity.headers().cacheControl();
    }
}
```

**新版本（Spring Security 6）：**
```java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.web.SecurityFilterChain;

@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**")
                        .allowedOriginPatterns("*")
                        .allowedMethods("*")
                        .allowedHeaders("*")
                        .allowCredentials(false)
                        .maxAge(3600);
            }
        };
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                .httpBasic(AbstractHttpConfigurer::disable)
                .csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/v1/xxxservice/**").hasAnyRole("ADMIN", "USER")
                        // SpringDoc OpenAPI endpoints
                        .requestMatchers("/swagger-ui/**", "/swagger-ui.html", "/v3/api-docs/**", "/v3/api-docs.yaml").permitAll()
                        .anyRequest().authenticated()
                )
                .addFilterBefore(new JWTFilter(), UsernamePasswordAuthenticationFilter.class);

        httpSecurity.headers(headers -> headers.cacheControl(cache -> {}));
        
        return httpSecurity.build();
    }
}
```

**需要移除的 import：**
```java
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import static org.springframework.web.cors.CorsConfiguration.ALL;
```

**需要添加的 import：**
```java
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.web.SecurityFilterChain;
```

---

### 4. javax → jakarta 命名空间迁移

| 旧包名 | 新包名 |
|--------|--------|
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.servlet.*` | `jakarta.servlet.*` |

**示例：**
```java
// 旧
import javax.persistence.*;
import javax.validation.Valid;
import javax.validation.constraints.NotNull;
import javax.servlet.http.HttpServletRequest;

// 新
import jakarta.persistence.*;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotNull;
import jakarta.servlet.http.HttpServletRequest;
```

---

### 5. Controller OpenAPI 注解（可选，用于生成更好的 SDK）

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;

@RestController
@RequestMapping("api/v1/xxxservice")
@Tag(name = "服务名称", description = "服务描述")
public class XxxController {

    @Operation(summary = "操作摘要", description = "操作详细描述")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "成功"),
        @ApiResponse(responseCode = "400", description = "请求错误")
    })
    @GetMapping("/xxx")
    public HttpEntity getXxx(
            @Parameter(description = "参数描述") @PathVariable String id,
            @RequestHeader HttpHeaders headers) {
        // ...
    }
}
```

---

### 6. Entity OpenAPI Schema 注解（可选）

```java
import io.swagger.v3.oas.annotations.media.Schema;

@Entity
@Schema(description = "实体描述")
public class XxxEntity {
    
    @Schema(description = "字段描述", example = "示例值")
    private String fieldName;
}
```

---

### 7. application.properties 配置更新

```properties
# 旧配置（移除）
swagger.controllerPackage=xxx.controller

# 新配置（可选）
springdoc.packages-to-scan=xxx.controller
springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
```

---

### 8. 快速查找需要修改的文件

```bash
# 查找使用 @EnableSwagger2 的文件
grep -r "@EnableSwagger2" --include="*.java" .

# 查找继承 WebSecurityConfigurerAdapter 的文件
grep -r "WebSecurityConfigurerAdapter" --include="*.java" .

# 查找 javax.persistence 导入
grep -r "import javax.persistence" --include="*.java" .

# 查找 javax.validation 导入
grep -r "import javax.validation" --include="*.java" .

# 查找 javax.servlet 导入  
grep -r "import javax.servlet" --include="*.java" .

# 查找旧版 swagger 配置
grep -r "swagger.controllerPackage" --include="*.properties" --include="*.yml" .
```

---

### 9. 验证升级

```bash
# 编译单个模块
mvn compile -pl ts-xxx-service -DskipTests

# 编译所有模块
mvn compile -DskipTests

# 启动服务后访问 OpenAPI 文档
# http://localhost:port/swagger-ui.html
# http://localhost:port/v3/api-docs (JSON)
# http://localhost:port/v3/api-docs.yaml (YAML)
```

---

### 10. 生成客户端 SDK

升级完成后，可以使用 OpenAPI Generator 生成客户端 SDK：

```bash
# 安装 openapi-generator
npm install @openapitools/openapi-generator-cli -g

# 生成 Java 客户端
openapi-generator-cli generate \
  -i http://localhost:port/v3/api-docs \
  -g java \
  -o ./generated-client

# 生成 TypeScript 客户端
openapi-generator-cli generate \
  -i http://localhost:port/v3/api-docs \
  -g typescript-axios \
  -o ./generated-client-ts
```

