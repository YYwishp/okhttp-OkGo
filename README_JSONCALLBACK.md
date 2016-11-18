 ![image](https://github.com/jeasonlzy/Screenshots/blob/master/okgo/logo4.jpg)

# JsonCallback自定义

* 一般服务端返回的数据格式都是这样的
```java
{
	"code":0,
	"msg":"请求成功",
	"data":{
		"id":123456,
		"name":"张三",
		"age":18
	}	
}
```

* 那么你需要定义的JavaBean有两种方式

第一种：将code和msg也定义在javabean中
```java
public class Login{
    public int code;
    public String msg;
    public ServerModel data;

    public class ServerModel{
        public long id;
        public String name;
        public int age;
    }
}
```

第二种：使用泛型，分离基础包装与实际数据，这样子需要定义两个javabean，一个全项目通用的`LzyResponse`，一个单纯的业务模块需要的数据
```java
public class LzyResponse<T> {
    public int code;
    public String msg;
    public T data;
}

public class ServerModel {
    public long id;
    public String name;
    public int age;
}
```

* 提供的demo中按照第二种方式实现的，对于第二种方式，我们在创建 JsonCallback 的时候，需要这么将`LzyResponse<ServerModel>`作为一个泛型传递，相当于传递了两层，泛型中包含了又一个泛型
```java
OkGo.get(Urls.URL_METHOD)//
    .tag(this)//
    .execute(new JsonCallback<LzyResponse<ServerModel>>(this) {
        @Override
        public void onSuccess(LzyResponse<ServerModel> responseData, Call call, Response response) {
            
        }
    });
```

* 那么在`JsonCallback`中对应的解析代码如下,详细的原理就不说了，下面的注释很详细
```java
@Override
public T convertSuccess(Response response) throws Exception {

    // 重要的事情说三遍，不同的业务，这里的代码逻辑都不一样，如果你不修改，那么基本不可用
    // 重要的事情说三遍，不同的业务，这里的代码逻辑都不一样，如果你不修改，那么基本不可用
    // 重要的事情说三遍，不同的业务，这里的代码逻辑都不一样，如果你不修改，那么基本不可用

    //以下代码是通过泛型解析实际参数,泛型必须传
    //这里为了方便理解，假如请求的代码按照上述注释文档中的请求来写，那么下面分别得到是

    //com.lzy.demo.callback.DialogCallback<com.lzy.demo.model.LzyResponse<com.lzy.demo.model.ServerModel>> 得到类的泛型，包括了泛型参数
    Type genType = getClass().getGenericSuperclass();
    //从上述的类中取出真实的泛型参数，有些类可能有多个泛型，所以是数值
    Type[] params = ((ParameterizedType) genType).getActualTypeArguments();
    //我们的示例代码中，只有一个泛型，所以取出第一个，得到如下结果
    //com.lzy.demo.model.LzyResponse<com.lzy.demo.model.ServerModel>
    Type type = params[0];
    
    // 这里这么写的原因是，我们需要保证上面我解析到的type泛型，仍然还具有一层参数化的泛型，也就是两层泛型
    // 如果你不喜欢这么写，不喜欢传递两层泛型，那么以下两行代码不用写，并且javabean按照第一种方式定义就可以实现
    if (!(type instanceof ParameterizedType)) throw new IllegalStateException("没有填写泛型参数");
    //如果确实还有泛型，那么我们需要取出真实的泛型，得到如下结果
    //class com.lzy.demo.model.LzyResponse
    //此时，rawType的类型实际上是 class，但 Class 实现了 Type 接口，所以我们用 Type 接收没有问题
    Type rawType = ((ParameterizedType) type).getRawType();

    //这里我们既然都已经拿到了泛型的真实类型，即对应的 class ，那么当然可以开始解析数据了，我们采用 Gson 解析
    //以下代码是根据泛型解析数据，返回对象，返回的对象自动以参数的形式传递到 onSuccess 中，可以直接使用
    JsonReader jsonReader = new JsonReader(response.body().charStream());
    if (rawType == Void.class) {
        //无数据类型,表示没有data数据的情况（以  new DialogCallback<LzyResponse<Void>>(this)  以这种形式传递的泛型)
        SimpleResponse baseWbgResponse = Convert.fromJson(jsonReader, SimpleResponse.class);
        response.close();
        //noinspection unchecked
        return (T) baseWbgResponse.toLzyResponse();
    } else if (rawType == LzyResponse.class) {
        //有数据类型，表示有data
        LzyResponse lzyResponse = Convert.fromJson(jsonReader, type);
        response.close();
        int code = lzyResponse.code;
        //这里的0是以下意思
        //一般来说服务器会和客户端约定一个数表示成功，其余的表示失败，这里根据实际情况修改
        if (code == 0) {
            //noinspection unchecked
            return (T) lzyResponse;
        } else if (code == 104) {
            //比如：用户授权信息无效，在此实现相应的逻辑，弹出对话或者跳转到其他页面等,该抛出错误，会在onError中回调。
            throw new IllegalStateException("用户授权信息无效");
        } else if (code == 105) {
            //比如：用户收取信息已过期，在此实现相应的逻辑，弹出对话或者跳转到其他页面等,该抛出错误，会在onError中回调。
            throw new IllegalStateException("用户收取信息已过期");
        } else if (code == 106) {
            //比如：用户账户被禁用，在此实现相应的逻辑，弹出对话或者跳转到其他页面等,该抛出错误，会在onError中回调。
            throw new IllegalStateException("用户账户被禁用");
        } else if (code == 300) {
            //比如：其他乱七八糟的等，在此实现相应的逻辑，弹出对话或者跳转到其他页面等,该抛出错误，会在onError中回调。
            throw new IllegalStateException("其他乱七八糟的等");
        } else {
            throw new IllegalStateException("错误代码：" + code + "，错误信息：" + lzyResponse.msg);
        }
    } else {
        response.close();
        throw new IllegalStateException("基类错误无法解析!");
    }
}
```
* JsonConvet 与 JsonCallback 一样，只是 JsonCallback 是在传统回调形式中用的，而 JsonConvert 是在 OkRx 中使用的，其余没有任何区别

### 更详细的代码请自行看 demo ，有很详细的注释