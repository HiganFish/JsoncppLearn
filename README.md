`Json::Value root;`
`root["action"] = "run";`
首先是 `[]` 运算符重载, 统一不同的重载类型
调用`resolveReference(char const* key, char const* end)`进行统一的添加k操作

首先会进行 校验参数, 然后将key封装成一个对象CZString (封装过程为将传入指针保存到 CZString对象中的cstr_)
封装完成后再保存k v的map<CZString, Value>中查找有无相同的CZString(ke)y有的话返回Value引用,
没有则创建新的<CZStrng, 空Value>存入map并返回Value的引用

然后是`=`运算符重载, "run"自动转换成Value对象
转换过程中, 通过`duplicateAndPrefixStringValue`将"run"进行了封装 
`char* string_; // if allocated_, ptr to { unsigned, char[] }.`
将长度封装到了一个char指针中, 有点类似自己设计tcp协议...

`=`函数将`[]`返回的对象引用 中的相关值替换成新的Value对象中的相关值


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
期初 使用默认构造函数的value type是nullxxx
然后每次调用`[]`的时候会修改掉 type 为 objectValue

```c++
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

1. 首先我很喜欢这种运算符重载的使用形式, 用起来非常的舒服, 需要重载两个运算符 `[]`和`=` 而且
他`=`重载使用的swap交换需要的属性, 感觉不错

2. 针对需要加载配置文件的类 使用了工厂模式
3. writeValue使用了递归处理.
4. 统一处理, k v都是Value对象
5. 将用户的string 拷贝到新的`char*`中 同时在开头增加了长度, 尾部补了0, 没有使用额外的变量去存储`char*`的长度
不太清楚这样做有什么好处
6. 常量全部用的 `static constexpr`修饰
7. 恰当的对象嵌套
8. 恰当的使用union将相关的属性再封装一层
