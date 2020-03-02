```c++
Json::Value root;
root["action"] = "run";
```
首先是 `[]运算符重载` , 统一不同的重载类型
调用`resolveReference()`进行统一的添加`k`操作

首先会进行 校验参数, 然后将key封装成一个对象CZString (封装过程为将传入指针保存到 CZString对象中的cstr_)
封装完成后再保存`k v`的`map<CZString, Value>`中查找有无相同的CZString(key)有的话返回Value引用,
没有则创建新的<CZStrng, 空Value>存入map并返回Value的引用

然后是`=运算符重载`, "run"自动转换成Value对象
转换过程中, 通过`duplicateAndPrefixStringValue()`将"run"进行了封装 
`char* string_; // if allocated_, ptr to { unsigned, char[] }.`
将长度封装到了一个char指针中, 有点类似自己设计tcp协议...

`=运算符重载`函数将`[]运算符重载`返回的对象引用 中的相关值`swap()`成新的Value对象中的相关值

到这里理解了在Jsoncpp中 一切都是Value 包括<K, V>键值对也是在Value对象中存储
每一个<k, v>都可以作为一个Value对象, 这样既实现了复杂嵌套Json中 v为对象的情况 妙啊

```c++
// 每个Value对象中都维护一个union
union ValueHolder {
    LargestInt int_;
    LargestUInt uint_;
    double real_;
    bool bool_;
    char* string_; // if allocated_, ptr to { unsigned, char[] }.

    // 将所有的存贮着key的CZString保存起来
	//  typedef std::map<CZString, Value> ObjectValues; // std::map<CZString, Value> 键值对
    ObjectValues* map_;
  } value_;
```

下面的switch的这个type 会在很多地方被修改掉.
起初 使用默认构造函数的value type是nullxxx
然后调用`[]运算符重载`的时候会修改掉 type 为 objectValue

```c++
// 递归调用 进行处理
void BuiltStyledStreamWriter::writeValue(Value const& value) {
  switch (value.type()) {
  case nullValue:
    pushValue(nullSymbol_);
    break;
  case intValue:
    pushValue(valueToString(value.asLargestInt()));
    break;
  case uintValue:
    pushValue(valueToString(value.asLargestUInt()));
    break;
  case realValue:
    pushValue(valueToString(value.asDouble(), useSpecialFloats_, precision_,
                            precisionType_));
    break;
  case stringValue: {
    // Is NULL is possible for value.string_? No.
    char const* str;
    char const* end;
    bool ok = value.getString(&str, &end);
    if (ok)
      pushValue(valueToQuotedStringN(str, static_cast<unsigned>(end - str),
                                     emitUTF8_));
    else
      pushValue("");
    break;
  }
  case booleanValue:
    pushValue(valueToString(value.asBool()));
    break;
  case arrayValue:
    writeArrayValue(value);
    break;
  case objectValue: {
    Value::Members members(value.getMemberNames());
    if (members.empty())
      pushValue("{}");
    else {
      writeWithIndent("{");
      indent();
      auto it = members.begin();
      for (;;) {
        String const& name = *it;
        Value const& childValue = value[name];
        writeCommentBeforeValue(childValue);
        writeWithIndent(valueToQuotedStringN(
            name.data(), static_cast<unsigned>(name.length()), emitUTF8_));

        // :
        *sout_ << colonSymbol_;
        writeValue(childValue);
        if (++it == members.end()) {
          writeCommentAfterValueOnSameLine(childValue);
          break;
        }
        *sout_ << ",";
        writeCommentAfterValueOnSameLine(childValue);
      }
      unindent();
      writeWithIndent("}");
    }
  } break;
  }
}
```

这次是根据下面的例子分析的
```c++
int main() {
  Json::Value root;
  Json::Value data;
  constexpr bool shouldUseOldWay = false;

  // 左侧返回Value的引用
  root["action"] = "run";
  data["number"] = 1;
  root["data"] = data;

  if (shouldUseOldWay) {
    Json::FastWriter writer;
    const std::string json_file = writer.write(root);
    std::cout << json_file << std::endl;
  } else {
    // 配置文件也是Value对象, 我用我自己.jpg
    Json::StreamWriterBuilder builder;
    const std::string json_file = Json::writeString(builder, root);
    std::cout << json_file << std::endl;
  }
  return EXIT_SUCCESS;
}
```

算是了解到了这个库的大概工作流程了. 能学到的部分 吃完午饭继续写.
吃完饭试了试家里的显示器, 家里的显示器还是太老了...... 八年的显示器了 看得我眼花
还是继续用笔记本吧

1. 首先我很喜欢这种运算符重载的使用形式, 用起来非常的舒服, 需要重载两个运算符 `[]`和`=` 而且
`=运算符`重载使用的swap交换需要的属性, 感觉不错
(后来我看了EffectiveC++ 发现这是`=运算符处理自我赋值`太巧了)

2. 针对需要加载配置文件的类 使用了工厂模式
3. writeValue使用了递归处理.
4. 统一处理, k v都是Value对象
5. 将用户的string 拷贝到新的`char*`中 同时在开头增加了长度, 尾部补了0, 没有使用额外的变量去存储`char*`的长度
不太清楚这样做有什么好处
6. 常量全部用的 `static constexpr`修饰
7. 恰当的对象嵌套


----

看完了从 Value到Json 接下来看看从Json到Value

```c++
int main() {
  const std::string rawJson = R"({"Age": 20, "Name": "colin"})";
  const auto rawJsonLength = static_cast<int>(rawJson.length());
  constexpr bool shouldUseOldWay = false;
  JSONCPP_STRING err;
  Json::Value root;

  if (shouldUseOldWay) {
    Json::Reader reader;
    reader.parse(rawJson, root);
  } else {
      // 默认构造函数 使用默认的配置
    Json::CharReaderBuilder builder;

    // 根据builder的配置生成CharReader类
    const std::unique_ptr<Json::CharReader> reader(builder.newCharReader());
    if (!reader->parse(rawJson.c_str(), rawJson.c_str() + rawJsonLength, &root,
                       &err)) {
      std::cout << "error" << std::endl;
      return EXIT_FAILURE;
    }
  }
  const std::string name = root["Name"].asString();
  const int age = root["Age"].asInt();

  std::cout << name << std::endl;
  std::cout << age << std::endl;
  return EXIT_SUCCESS;
}
```

解析的关键在于`parse()`函数 传入字符串的首尾指针, 和一个Value引用
进入函数后将传入的变量保存到了自己的成员变量中.

慢慢发现 这个库将很多的默认类型 起了别名 可能是为了统一类型 类似UE4也是自己搞了一套

主要是`OurReader`这个负责解析
```c++
// 使用了大量的using
using Char = char;
using Location = const Char*;
```

解析逻辑就是`parse()`调用`readValue()`
`readValue()`负责 获取下一次数据类型type_ ->switch(type_) 根据分支决定是否递归再次调用`readValue`
总算把逻辑看懂了, 代码依然认为很赞
`readToken()`这个函数贯穿整个解析, 通过这个函数获取到下一次的数据是什么类型, 同时移动相关的状态指针
`readToken()` 内部大量调用下面的函数 修改当前指针 同时返回字符, 然后switch这个字符判断下一次的数据类型
比如遇到`{`就是一个对象的开始设置好type并返回 遇到`}`就是对象的结束.....
```c++
// 获取current_指向的字符 并自增
OurReader::Char OurReader::getNextChar()
{
  if (current_ == end_)
    return 0;
  return *current_++;
}
```

一般第一次调用type_会被设置为对象类型, 然后`readValue()`进入`case 对象分支`
`case对象分支中`
先进行了一次`readToken()`获取到了k的类型, 然后根据k类型分支 将k的值转换成对应的value
之后又是一次`readToken()`判断是否存在`:`不存在就是错误
在之后使用这个方法, 保存`k`的vallue获取`v`的value再次调用`readValue()`填充值 返回后继续走
```c++
// 这里保存了name 这个k 将name的 v放入了顶层
Value& value = currentValue()[name];
nodes_.push(&value);
bool ok = readValue();
```
读取完`v`的value之后必定是`,`或者`}`又是一次判断 成功判断后一个`k v`就获取完毕了

` using Nodes = std::stack<Value*>`
后面解析的代码更加的妙不可言, 使用`Nodes nodes_{}`存储当前的value 实现函数之间的操作

代码合理的组织
比如`case 对象分支`必定是一个`k`一个`:`一个`v` 然后一个分隔符`,`或`}` 这些放入了一个函数

然后获取到`k`之后使用`Value& value = currentValue()[name]`获取`v`

最后依然是递归的使用