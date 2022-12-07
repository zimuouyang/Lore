| 类别       | 关键字 | 返回类型       | 搭档                  |
| ---------- | ------ | -------------- | --------------------- |
| 多元素同步 | sync*  | `Iterable<T> ` | yield、yield*         |
| 单元素异步 | async  | `Future<T> `   | await                 |
| 多元素异步 | async* | `Stream<T> `   | yield、yield* 、await |

#### 一、多元素同步函数生成器

##### 1. `sync*` 和 `yield`

> `sync*`是一个dart语法`关键字`。`它标注在函数{ 之前，其方法必须返回一个 Iterable<T>对象`
>  👿 的码为`\u{1f47f}`。下面是使用`sync*`生成后10个emoji`迭代(Iterable)对象`的方法

```dart
main() {
  getEmoji(10).forEach(print);
}

Iterable<String> getEmoji(int count) sync* {
  Runes first = Runes('\u{1f47f}');
  for (int i = 0; i < count; i++) {
    yield String.fromCharCodes(first.map((e) => e + i));
  }
}
```

##### 2、`sync*` 和 `yield*`

> yield*`又是何许人也? 记住一点`yield*`后面的表达式是一个`Iterable<T>对象`
>  比如下面`getEmoji`方法是核心，现在想要打印每次的时间，使用`getEmojiWithTime` `yield*`之后的`getEmoji(count).map((e)...`便是一个可迭代对象`Iterable<String>

```dart
main() {
  getEmojiWithTime(10).forEach(print);
}

Iterable<String> getEmojiWithTime(int count) sync* {
  yield* getEmoji(count).map((e) => '$e -- ${DateTime.now().toIso8601String()}');
}

Iterable<String> getEmoji(int count) sync* {
  Runes first = Runes('\u{1f47f}');
  for (int i = 0; i < count; i++) {
    yield String.fromCharCodes(first.map((e) => e + i));
  }
}
```

#### 二、异步处理: `async`和`await`

> `async`是一个dart语法`关键字`。`它标注在函数{ 之前，其方法必须返回一个 Future<T>对象`
>  对于耗时操作，通常用`Future<T>`对象异步处理，下面`fetchEmoji方法`模拟2s加载耗时

```dart
main() {
  print('程序开启--${DateTime.now().toIso8601String()}');
  fetchEmoji(1).then(print);
}

Future<String> fetchEmoji(int count) async{
  Runes first = Runes('\u{1f47f}');
  await Future.delayed(Duration(seconds: 2));//模拟耗时
  print('加载结束--${DateTime.now().toIso8601String()}');
  return String.fromCharCodes(first.map((e) => e + count));
}
```

#### 三、多元素异步函数生成器:

##### 1.`async*`和`yield`、`await`

> `async*`是一个dart语法`关键字`。`它标注在函数{ 之前，其方法必须返回一个 Stream<T>对象`
>  下面`fetchEmojis`被`async*`标注，所以返回的必然是`Stream`对象
>  注意`被async*`标注的函数，可以在其内部使用`yield、yield*、await`关键字

```dart
main() {
  fetchEmojis(10).listen(print);
}

Stream<String> fetchEmojis(int count) async*{
  for (int i = 0; i < count; i++) {
    yield await fetchEmoji(i);
  }
}

Future<String> fetchEmoji(int count) async{
  Runes first = Runes('\u{1f47f}');
  print('加载开始--${DateTime.now().toIso8601String()}');
  await Future.delayed(Duration(seconds: 2));//模拟耗时
  print('加载结束--${DateTime.now().toIso8601String()}');
  return String.fromCharCodes(first.map((e) => e + count));
}
```

##### 2.`async*`和`yield*`、`await`

> 和上面的`yield*`同理，`async*`方法内使用`yield*`,其后对象必须是`Stream<T>`对象
>  如下`getEmojiWithTime`对`fetchEmojis`流进行map转换，前面需要加`yield*`

```dart
main() {
  getEmojiWithTime(10).listen(print);
}

Stream<String> getEmojiWithTime(int count) async* {
  yield* fetchEmojis(count).map((e) => '$e -- ${DateTime.now().toIso8601String()}');
}

Stream<String> fetchEmojis(int count) async*{
  for (int i = 0; i < count; i++) {
    yield await fetchEmoji(i);
  }
}

Future<String> fetchEmoji(int count) async{
  Runes first = Runes('\u{1f47f}');
  await Future.delayed(Duration(seconds: 2));//模拟耗时
  return String.fromCharCodes(first.map((e) => e + count));
}
```

