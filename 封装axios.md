# 一、axios二次封装

## 1. 简单的封装

```js
import axios from 'axios'
import { Loading, Message } from 'element-ui'; //引入UI框架

//对axios的简单封装
export const request = (url,method,params,callback) => {
  //打开弹窗
  const lxlLoading = Loading.service({
    text: '拼命加载中......',
    background: 'rgba(0, 0, 0, 0.8)',
    spinner: 'el-icon-loading'
  })
  //发送axios请求
  _axios.request(url,{
    method: method,
    params: params
  }).then(response => {
      if(response.data.code === 200)
      //有内容输出内容
      if (response.data.message){
			Message.success(response.data.message)
		}
		callback(response.data.content);//返回数据
  }).catch(error => {
    alert(error)//抛出异常信息
  }).finally(() =>{
    //关闭弹窗
    lxlLoading.close()
  })
}

//再自己单独定义一下get和post请求
export const get = (url,params,callback) => {
	request(url, 'get', params, callback)
}

export const post = (url,params,callback) => {
	request(url, 'post', params, callback)
}
```

## 2. 表单post请求的封装

```js
export const lxlaxios = _axios
export const request = (url,method,params,callback) => {
// 加载组件
const lxlLoading = Loading.service({
	text: '加载中......'
})
const lxlconfig = {
	method: method,
}
if (method === 'get') {
	lxlconfig.params = params
}else{
	const formData = new FormData()
	for (const key in params) {
		if (params[key] instanceof Array) {
			for (let i = 0; i < params[key].length; i++) {
				formData.append(key,params[key][i])
			}
		} else {
			formData.append(key,params[key])
		}
	}
	lxlconfig.data = formData
}
_axios.request(url,lxlconfig
	).then(response => {
	if (response.data.code === 200){
		if (response.data.message){
			Message.success(response.data.message)
		}
		callback(response.data.content);
	} else {
		Message.error(response.data.message)
	}
	}).catch(error => {
		Message.error(error.message)
	}).finally(() => {
		lxlLoading.close()
	})
}
export const get = (url,params,callback) => {
	request(url, 'get', params, callback)
}

export const post = (url,params,callback) => {
	request(url, 'post', params, callback)
}
```

# 二、登录功能的实现

## 方式一：vuex持久化+jwt

* 前端
  * 在store中的vuex定义全局的token（接受并存储）
  * 在router中定义白名单和转发校验（验证token）
  * 引入vuex持久化插件，讲token保存在sessionStorage中（关闭即失效，待解决...）

* 后端
  * 引入jwt令牌依赖
  * 编写jwtUtils工具类
  * 在用户登录校验通过后，使用工具类创造个token返回给前端


## 1.后端

```java
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
</dependency>
    
/**
 * @Description TODO
 * @Date 2022/4/28 23:09
 * @Version 1.0
 */
public class JwtUtils {
    private static final String KEY = "lxl";

    //加密
    public static String create(User user){
        return JWT.create().withClaim("tokenId",user.getId())
                .withClaim("tokenName",user.getName())
                .sign(Algorithm.HMAC256(KEY));
    }

    //解密
    public static User decode(String token) throws MyException {
        try{
            DecodedJWT verify = JWT.require(Algorithm.HMAC256(KEY)).build().verify(token);
            Integer tokenId = verify.getClaim("tokenId").asInt();
            String tokenName = verify.getClaim("tokenName").asString();
            User user = new User();
            user.setId(tokenId);
            user.setName(tokenName);
            return user;
        }catch (Exception e){
            e.printStackTrace();
            throw new MyException("用户未登录");
        }

    }
}

贼impl里写login的逻辑
    @Override
    public String login(String username, String password) {
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        wrapper.eq("username",username);
        User user = this.getOne(wrapper);
        if (user != null && passwordEncoder.matches(password,user.getPassword())){
            return JwtUtils.create(user);
        }
        return null;
    }
```

## 2.前端

* 添加vuex持久化存储，不然每次刷新都会丢失token

```js
import createPersistedState from "vuex-persistedstate";//这是引入持久化插件
//这是定义存储位置
plugins: [
      createPersistedState({
        storage: window.sessionStorage
      })
  ],
```



### 2.1定义全局的token

```js
import Vue from 'vue'
import Vuex from 'vuex'
import createPersistedState from "vuex-persistedstate";//这是引入持久化插件

Vue.use(Vuex)

export default new Vuex.Store({
  plugins: [
      createPersistedState({
        storage: window.sessionStorage
      })
  ],
  state: {
    //定义全局变量
    state: null
  },
  getters: {
    GET_TOKEN (state) { return state.token }
  },
  mutations: {
    SET_TOKEN (state, token) {
      state.token = token
    }
  },
  actions: {
  },
  modules: {
  }
})
```

### 2.2定义白名单和转发校验

```js
//定义白名单
const white = ['lxlLogin']
//转发前校验
router.beforeEach((to,from,next) => {
  for (const index in white) {
    if(white[index] === to.name){
      next();
      return
    }
  }
  if (store.getters.GET_TOKEN){
    next()
    return
  }
  next('/login')
})
```

* 发送请求

```js
request('http://localhost:8080/user-center/user/login','post',this.form,response=>{
  this.$store.commit('SET_TOKEN',response)//去存token
  this.$router.push('/')//去转发首页
})
```

## 方式二：服务器缓存session(tomcat)

* 在后端中设置HttpServletRequest request 参数
* 在校验通过后可以将user信息设置到服务器端的session缓存中

```java
@PostMapping("/login")
    public User userLogin(UserLoginRequest userLoginRequest, HttpServletRequest request){
        System.out.println("我进入此方法了");
        if (userLoginRequest == null){
            return null;//传来的用户名和密码为空，就登录返回错误
        }
        String username = userLoginRequest.getUsername();
        String password = userLoginRequest.getPassword();
        if (StringUtils.isAnyBlank(username, password)){
            return null;//二者为空也返回错误
        }
        return userService.userLogin(username, password, request);//service具体查询数据库逻辑自己写
    }
```

* 设置session（在service写，我只写了部分的）

```java
USER_LOGIN_STATE ---》 定义的final常量
   
Object user = request.getSession().setAttribute("USER_LOGIN_STATE",safetyUser);
if (user == null){
    return ResultJson.failed("登录失败");
}
return ResultJson.success("请求成功");
```

* 可以在controller中写个是否为管理员(小功能)

```java
/**
     * 是否为管理员
     * @param request
     * @return
     */
private boolean isAdmin(HttpServletRequest request){
    //管理员权限
    User user = (User) request.getSession().getAttribute(USER_LOGIN_STATE);
    if (user == null || user.getUserRole() != ADMIN_ROLE){
        return false; 	ADMIN_ROLE是定义的常量类constant
    }
    return true;
}
```

* 请求其他接口时，注意查询session是否存在

```java
@RequestMapping("/test")
public ResultJson<String> test(HttpServletRequest request){
    Object user = request.getSession().getAttribute("USER_LOGIN_STATE");
    if (user == null){
        return ResultJson.failed("未登录");
    }
    return ResultJson.success("请求成功");
}
```

* 后端yml中设置session的过期时间

```yam
server:
  servlet:
    session:
      timeout: 60 #一分钟过期，如果小于1分钟，sprignboot会默认设置1分钟
```

* 总结

```xml
1.浏览器只要一关闭就会清除session，需要二次登录
2.如果一直在访问改网站，就不会删除session，而且会自动向后增加时间
```

## 方式三： redis+vuex+jwt

* springboot 整合 redis

待...

# 三、docker部署

* 后端Dockerfile

```xml
FROM maven:3.5-jdk-8-alpine as builder #从中央pull需要支持项目的镜像
FROM openjdk:8 #用这个

# Copy local code to the container image.
WORKDIR /app 	#项目工作目录
COPY pom.xml . 	#复制pom.xml到app目录中
COPY src ./src 	#复制src到app/src目录中

# Build a release artifact.
RUN mvn package -DskipTests 	#执行maven打包命令

# Run the web service on container startup.
CMD ["java","-jar","/app/target/user-center-backend-0.0.1-SNAPSHOT.jar","--spring.profiles.active=prod"]
```

* 前端Dockerfile

```xml
FROM nginx

WORKDIR /usr/share/nginx/html/ #工作目录
USER root

COPY ./docker/nginx.conf /etc/nginx/conf.d/default.conf # /docker/nginx.conf 这个就是把前端整体目录中的/docker/nginx.conf，所以要创建个docker目录，将nginx.conf放在里面

COPY ./dist  /usr/share/nginx/html/ 	#将前端打包的dist目录下的东西，映射到nginx下html目录

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]  #构建后自己run进行启动
```

* nginx.conf

```xml
server {
    listen 80;

    # gzip config
    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 9;
    gzip_types text/plain text/css text/javascript application/json application/javascript application/x-javascript application/xml;
    gzip_vary on;
    gzip_disable "MSIE [1-6]\.";

    root /usr/share/nginx/html;
    include /etc/nginx/mime.types;

    location / {
        try_files $uri /index.html;  #配合vue路由去找静态资源
    }

}
```

* docker build

```xml
# 后端 - 构建镜像，这最后一个点 . 是当前目录下的 Dockerfile
docker build -t user-center-backend:v0.0.1 .

# 前端
docker build -t user-center-front:v0.0.1 .
```

* docker 启动

```xml
# 前端
docker run -p 80:80 -d user-center-front:v0.0.1

# 后端
docker run -p 8080:8080 -d user-center-backend:v0.0.1
```

