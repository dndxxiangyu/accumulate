## 背景

现在项目开发过程存在一个问题：后端和移动端的开发进度是并行。我理解的正常开发流程：

- prd评审
- 设计稿评审
- 后端进入开发，提供可测试接口。
- 移动端进入开发，针对当前开发接口进行功能+连跳。

当然每个公司甚至不同的部门都会有不同的开发流程，为了能解决当前开发流程中的痛点，mock服务看去可以有效解决前后端开发步调不协调问题。



## 选型

其实个人在做选型的时候，是会有比较大的倾向性的，比如我可能对ruby熟悉一点，加上rails能够快速开发web应用，所以自然就想到了通过rails来开发。

这块可能比较灵活，需要owner去做适合自己的开发决策。



## 开发

1. 搭建环境：ruby一般使用RVM来管理他的环境，安装rails
2. 创建rails工程：rails new myMock，会创建mock项目工程。

3. 由于rails是典型的mvc架构，所以在增加一个view之前，需要同时增加的有：controller和mode，由于mock工程不涉及到mode这快，所以我们可以直接生成controller和view。

## 实现

1. 修改routes.rb中的内容， 响应url，

```ruby
Rails.application.routes.draw do
  post 'login/create', to: 'login#create'
end
```

2. 生成解析该router的controller，然后在create方法中，返回json或者具体数据格式。

```ruby
class LoginController < ApplicationController
  # 避免csrf攻击
  protect_from_forgery with: :null_session
  def index
  end

  def login_type
    render json: {"hello": "world"}
  end
  def create
    render json: {hello: 'world'}
  end
end
```

3. 执行 bin/rails server，启动服务，然后在postman或浏览器中访问即可。



总体而言，快速搭建一个mock服务，rails还是很方便开发的。

## 参考：

[Rails入门](https://ruby-china.github.io/rails-guides/getting_started.html)

