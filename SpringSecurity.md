* `SecurityAutoConfiguration` 自动配置该类
* `SecurityFilterAutoConfiguration` 自动配置过滤器(在`SecurityAutoConfiguration`之后)
`AbstractConfiguredSecurityBuilder` 创建类
* 初始化过程
1 > 在初始化WebSecurity中(WebSecurityConfigurerAdapter#init)创建HttpSecurity,在WebSecurity#performBuild时进行HttpSecurity的初始化操作
