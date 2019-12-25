# detekt扩展RuleSets

## 现状

detekt是kotlin的静态代码分析框架，由于业务需要，所以需要在当前框架现有的基础上进行扩展。官方也有相[demo](https://github.com/arturbosch/detekt/tree/master/detekt-sample-extensions)，由于官方文档在搭建环境这块有些不清晰，所以通过搭建后进行记录。

## 操作

1. AS中创建jar module：wlib，detekt的代码检测规则的具体实现比如（TooManyFunctions），他是通过ServiceLoader来进行被加载。所以对应具体的Rule，需要在resources/META-INF/services/io.gitlab.arturbosch.detekt.api.RuleSetProvider进行注册。

2. RuleSetProvider中什么当前Provider可以关联的Rules，一个provider就是同一种Rule类型的组合。包括了ruleIdName，rule list。

3. 想要自定义Rule能够被detekt识别，需要和config.yml中的detekt默认实现Rules一样，需要在config.yml中定义。
4. 依赖detekt-formatting，依赖wlib模块，并且在执行gradle detekt之前，需要wlib的jar包。

5. 详情见commitid：3d590354e092dd2b2fdcfafed970eb80d1f77701 in :[MyDemo](https://github.com/dndxxiangyu/MyDemo/tree/master)

## 执行

当前RuleSerProvider的名称是：customer，当执行gradle后在终端中看到Rules：costomer，表示环境已经搭建完毕。

## 遇到问题

wlib打包问题，AS默认生成的jar module，gradle build过程中生成的jar包，默认不包含kt代码，这一块也困扰我好久。其实要看当前中定义Rules是不是真的生效了，直接解包是最有效的方式。

