### 问题背景：

> Springboot 启用拦截器后，Swagger无法访问

---

### 原因

   拦截器拦截了所有的请求，导致swagger也被拦截，当在进行鉴权的的时候，可能需要传入一些特定的参数，或者请求头信息，这样我们就无法正常通过swagger了。



### 解决

  配置静态资源处理器，以及将swagger的访问路径排除在外，即可解决问题。

```java
import com.eechain.sso.interceptor.AuthInterceptor;
import com.eechain.sso.interceptor.ResponseInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
@Configuration
public class SpringConfig implements WebMvcConfigurer {

  private AuthInterceptor authInterceptor; 
  @Autowired
  public void setAuthInterceptor(AuthInterceptor authInterceptor) {
    this.authInterceptor = authInterceptor;
  }

  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry
        .addInterceptor(authInterceptor)
        .addPathPatterns("/**")
        .excludePathPatterns("/account/**")
        // 静态资源
        .excludePathPatterns("/js/**", "/css/**", "/images/**", "/lib/**",
            "/fonts/**")
        // swagger-ui
        .excludePathPatterns("/swagger-resources/**", "/webjars/**",
            "/v2/**", "/swagger-ui.html/**");
    registry.addInterceptor(responseInterceptor).addPathPatterns("/**");
  }

  // 必须添加
  @Override
  public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("swagger-ui.html")
        .addResourceLocations("classpath:/META-INF/resources/");
    registry.addResourceHandler("/webjars/**")
        .addResourceLocations("classpath:/META-INF/resources/webjars/");

  }
}
```



