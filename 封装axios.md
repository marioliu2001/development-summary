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





