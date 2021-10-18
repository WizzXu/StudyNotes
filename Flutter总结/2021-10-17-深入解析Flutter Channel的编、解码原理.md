# 深入解析Flutter Channel的编解码原理
## 1. 简介
我们在进行Flutter混合开发过程中，一般会用到平台特定功能，这个时候就要用到 Channel 通信了。
Channel 是 Flutter 端与原生端制定的通信机制，用于 Dart 和平台之间的相互通信。

##### Channel 类型分为以下 3 种
（1）BaseMessageChannel ：用于传递字符串和半结构化的信息（在大内存数据块传递的情况下使用）
（2）MethodChannel：用于传递方法调用（Method Invocation）  
（3）EventChannel: 用于数据流（Event Streams）的通信
##### 消息使用平台通道在客户端（UI）和宿主（平台）之间传递，如下图所示：
![https://flutter.cn/assets/images/docs/PlatformChannels.png](https://flutter.cn/assets/images/docs/PlatformChannels.png)

## 2. 编码和解码
Channel的实现主要分为两部分
- 数据的传递
- 数据的编码和解码



**数据的传递** 本期我们先不做探索，网上的文章也很多，有需要的小伙伴可以关注我的后续文章。  
数据从Dart到原生端传递的过程核心原理就是 `Dart(编码/) -> C/C++ -> 原生端（解码）`  

### 2.1 Dart端的编码和解码
#### 2.1.1 三种类型Channel源码
##### BasicMessageChannel
```
class BasicMessageChannel<T> {
  /// Creates a [BasicMessageChannel] with the specified [name], [codec] and [binaryMessenger].
  ///
  /// The [name] and [codec] arguments cannot be null. The default [ServicesBinding.defaultBinaryMessenger]
  /// instance is used if [binaryMessenger] is null.
  /// 1. 创建 BasicMessageChannel 的时候需要传入 name 和 codec
  const BasicMessageChannel(this.name, this.codec, { BinaryMessenger? binaryMessenger })
      : assert(name != null),
        assert(codec != null),
        _binaryMessenger = binaryMessenger;

  /// The logical channel on which communication happens, not null.
  final String name;

  /// The message codec used by this channel, not null.
  final MessageCodec<T> codec;
  ...
}
```
创建 BasicMessageChannel 的时候需要传入 name 和 codec，其中 codec 没有默认值  
通常我们不使用 BasicMessageChannel

##### MethodChannel
```
class MethodChannel {
  /// Creates a [MethodChannel] with the specified [name].
  ///
  /// The [codec] used will be [StandardMethodCodec], unless otherwise
  /// specified.
  ///
  /// The [name] and [codec] arguments cannot be null. The default [ServicesBinding.defaultBinaryMessenger]
  /// instance is used if [binaryMessenger] is null.
  /// 1. 创建 MethodChannel 的时候可以传入 codec， 默认为 StandardMethodCodec
  const MethodChannel(this.name, [this.codec = const StandardMethodCodec(), BinaryMessenger? binaryMessenger ])
      : assert(name != null),
        assert(codec != null),
        _binaryMessenger = binaryMessenger;

  /// The logical channel on which communication happens, not null.
  final String name;

  /// The message codec used by this channel, not null.
  final MethodCodec codec;
  ...
}
```
创建 MethodChannel 的时候可以传入 codec， 默认为 StandardMethodCodec，StandardMethodCodec 分析 --> [StandardMethodCodec](#StandardMethodCodec_Dart)   
我们使用最多的就是 MethodChannel 

##### EventChannel
```
class EventChannel {
  /// Creates an [EventChannel] with the specified [name].
  ///
  /// The [codec] used will be [StandardMethodCodec], unless otherwise
  /// specified.
  ///
  /// Neither [name] nor [codec] may be null. The default [ServicesBinding.defaultBinaryMessenger]
  /// instance is used if [binaryMessenger] is null.
  /// 1. 创建 EventChannel 的时候可以传入 codec， 默认为 StandardMethodCodec
  const EventChannel(this.name, [this.codec = const StandardMethodCodec(), BinaryMessenger? binaryMessenger])
      : assert(name != null),
        assert(codec != null),
        _binaryMessenger = binaryMessenger;

  /// The logical channel on which communication happens, not null.
  final String name;

  /// The message codec used by this channel, not null.
  final MethodCodec codec;
  ...
```
创建 EventChannel 的时候可以传入 codec， 默认为 StandardMethodCodec，StandardMethodCodec 分析 --> [StandardMethodCodec](#StandardMethodCodec_Dart)    
我们在部分场景下会使用到 EventChannel

#### 2.1.2 <span id="StandardMethodCodec_Dart">StandardMethodCodec</span>
##### MessageCodec
所有编码器的父类是 MessageCodec，它是一个抽象类，里面定义了两个方法。编码和解码
```
abstract class MessageCodec<T> {
  /// Encodes the specified [message] in binary.
  ///
  /// Returns null if the message is null.
  ByteData? encodeMessage(T message);

  /// Decodes the specified [message] from binary.
  ///
  /// Returns null if the message is null.
  T? decodeMessage(ByteData? message);
}
```
##### StandardMessageCodec 
```
class StandardMessageCodec implements MessageCodec<Object?> {
  /// Creates a [MessageCodec] using the Flutter standard binary encoding.
  const StandardMessageCodec();

  static const int _valueNull = 0;
  static const int _valueTrue = 1;
  static const int _valueFalse = 2;
  static const int _valueInt32 = 3;
  static const int _valueInt64 = 4;
  static const int _valueLargeInt = 5;
  static const int _valueFloat64 = 6;
  static const int _valueString = 7;
  static const int _valueUint8List = 8;
  static const int _valueInt32List = 9;
  static const int _valueInt64List = 10;
  static const int _valueFloat64List = 11;
  static const int _valueList = 12;
  static const int _valueMap = 13;

  @override
  ByteData? encodeMessage(Object? message) {
    if (message == null)
      return null;
    final WriteBuffer buffer = WriteBuffer();
    writeValue(buffer, message);
    return buffer.done();
  }

  @override
  dynamic decodeMessage(ByteData? message) {
    if (message == null)
      return null;
    final ReadBuffer buffer = ReadBuffer(message);
    final Object? result = readValue(buffer);
    if (buffer.hasRemaining)
      throw const FormatException('Message corrupted');
    return result;
  }
}
```
首先定义了13种支持的数据类型，用int值表示。然后分别实现了编码和解码的方法
##### encodeMessage
```
ByteData? encodeMessage(Object? message) {
  if (message == null)
    return null;
  final WriteBuffer buffer = WriteBuffer();
  writeValue(buffer, message);
  return buffer.done();
}
```
```
void writeValue(WriteBuffer buffer, Object? value) {
  if (value == null) {
    buffer.putUint8(_valueNull);
  } else if (value is bool) {
    buffer.putUint8(value ? _valueTrue : _valueFalse);
  } else if (value is double) {  // Double precedes int because in JS everything is a double.
                                 // Therefore in JS, both `is int` and `is double` always
                                 // return `true`. If we check int first, we'll end up treating
                                 // all numbers as ints and attempt the int32/int64 conversion,
                                 // which is wrong. This precedence rule is irrelevant when
                                 // decoding because we use tags to detect the type of value.
    buffer.putUint8(_valueFloat64);
    buffer.putFloat64(value);
  } else if (value is int) {
    if (-0x7fffffff - 1 <= value && value <= 0x7fffffff) {
      buffer.putUint8(_valueInt32);
      buffer.putInt32(value);
    } else {
      buffer.putUint8(_valueInt64);
      buffer.putInt64(value);
    }
  } else if (value is String) {
    buffer.putUint8(_valueString);
    final Uint8List bytes = utf8.encoder.convert(value);
    writeSize(buffer, bytes.length);
    buffer.putUint8List(bytes);
  } else if (value is Uint8List) {
    buffer.putUint8(_valueUint8List);
    writeSize(buffer, value.length);
    buffer.putUint8List(value);
  } else if (value is Int32List) {
    buffer.putUint8(_valueInt32List);
    writeSize(buffer, value.length);
    buffer.putInt32List(value);
  } else if (value is Int64List) {
    buffer.putUint8(_valueInt64List);
    writeSize(buffer, value.length);
    buffer.putInt64List(value);
  } else if (value is Float64List) {
    buffer.putUint8(_valueFloat64List);
    writeSize(buffer, value.length);
    buffer.putFloat64List(value);
  } else if (value is List) {
    buffer.putUint8(_valueList);
    writeSize(buffer, value.length);
    for (final Object? item in value) {
      writeValue(buffer, item);
    }
  } else if (value is Map) {
    buffer.putUint8(_valueMap);
    writeSize(buffer, value.length);
    value.forEach((Object? key, Object? value) {
      writeValue(buffer, key);
      writeValue(buffer, value);
    });
  } else {
    throw ArgumentError.value(value);
  }
}
```
```
void writeSize(WriteBuffer buffer, int value) {
  assert(0 <= value && value <= 0xffffffff);
  if (value < 254) {
    buffer.putUint8(value);
  } else if (value <= 0xffff) {
    buffer.putUint8(254);
    buffer.putUint16(value);
  } else {
    buffer.putUint8(255);
    buffer.putUint32(value);
  }
}
```
编码过程
1. 首先定义一个字节缓冲区，用于存放数据编码成的二进制数据
2. 判断传入的对象的类型，是否在定义的13种数据类型之内，如果不在的话抛出`ArgumentError.value(value)`异常
3. 开始对数据进行编码。  
这些数据分为两种，  
一种是固定数据类型，这种数据长度固定。另一种是不固定数据类型，这种数据长度不固定，需要再次编码或者需要附加其他参数进行编码。  
##### decodeMessage
```
@override
dynamic decodeMessage(ByteData? message) {
  if (message == null)
    return null;
  final ReadBuffer buffer = ReadBuffer(message);
  final Object? result = readValue(buffer);
  if (buffer.hasRemaining)
    throw const FormatException('Message corrupted');
  return result;
}
```
```
/// Reads a value from [buffer] as written by [writeValue].
///
/// This method is intended for use by subclasses overriding
/// [readValueOfType].
Object? readValue(ReadBuffer buffer) {
  if (!buffer.hasRemaining)
    throw const FormatException('Message corrupted');
  final int type = buffer.getUint8();
  return readValueOfType(type, buffer);
}

/// Reads a value of the indicated [type] from [buffer].
///
/// The codec can be extended by overriding this method, calling super for
/// types that the extension does not handle. See the discussion at
/// [writeValue].
Object? readValueOfType(int type, ReadBuffer buffer) {
  switch (type) {
    case _valueNull:
      return null;
    case _valueTrue:
      return true;
    case _valueFalse:
      return false;
    case _valueInt32:
      return buffer.getInt32();
    case _valueInt64:
      return buffer.getInt64();
    case _valueFloat64:
      return buffer.getFloat64();
    case _valueLargeInt:
    case _valueString:
      final int length = readSize(buffer);
      return utf8.decoder.convert(buffer.getUint8List(length));
    case _valueUint8List:
      final int length = readSize(buffer);
      return buffer.getUint8List(length);
    case _valueInt32List:
      final int length = readSize(buffer);
      return buffer.getInt32List(length);
    case _valueInt64List:
      final int length = readSize(buffer);
      return buffer.getInt64List(length);
    case _valueFloat64List:
      final int length = readSize(buffer);
      return buffer.getFloat64List(length);
    case _valueList:
      final int length = readSize(buffer);
      final List<Object?> result = List<Object?>.filled(length, null, growable: false);
      for (int i = 0; i < length; i++)
        result[i] = readValue(buffer);
      return result;
    case _valueMap:
      final int length = readSize(buffer);
      final Map<Object?, Object?> result = <Object?, Object?>{};
      for (int i = 0; i < length; i++)
        result[readValue(buffer)] = readValue(buffer);
      return result;
    default: throw const FormatException('Message corrupted');
  }
}
```
解码过程
1. 首先读取一位int值，去定类型标识
2. 根据读取的类型标识，分别读取后面的数据

### 2.2 Java端的编码和解码
#### 2.2.1 三种类型Channel源码
##### BasicMessageChannel
```
public final class BasicMessageChannel<T> {
  private static final String TAG = "BasicMessageChannel#";
  public static final String CHANNEL_BUFFERS_CHANNEL = "dev.flutter/channel-buffers";

  @NonNull private final BinaryMessenger messenger;
  @NonNull private final String name;
  @NonNull private final MessageCodec<T> codec;

  /**
   * Creates a new channel associated with the specified {@link BinaryMessenger} and with the
   * specified name and {@link MessageCodec}.
   *
   * @param messenger a {@link BinaryMessenger}.
   * @param name a channel name String.
   * @param codec a {@link MessageCodec}.
   * 1. 创建 BasicMessageChannel 的时候需要传入 name 和 codec
   */
  public BasicMessageChannel(
      @NonNull BinaryMessenger messenger, @NonNull String name, @NonNull MessageCodec<T> codec) {
    if (BuildConfig.DEBUG) {
      if (messenger == null) {
        Log.e(TAG, "Parameter messenger must not be null.");
      }
      if (name == null) {
        Log.e(TAG, "Parameter name must not be null.");
      }
      if (codec == null) {
        Log.e(TAG, "Parameter codec must not be null.");
      }
    }
    this.messenger = messenger;
    this.name = name;
    this.codec = codec;
  }
  ...
}
```
创建 BasicMessageChannel 的时候需要传入 name 和 codec，其中 codec 没有默认值  
通常我们不使用 BasicMessageChannel

##### MethodChannel
```
public class MethodChannel {
  private static final String TAG = "MethodChannel#";

  private final BinaryMessenger messenger;
  private final String name;
  private final MethodCodec codec;

  /**
   * Creates a new channel associated with the specified {@link BinaryMessenger} and with the
   * specified name and the standard {@link MethodCodec}.
   *
   * @param messenger a {@link BinaryMessenger}.
   * @param name a channel name String.
   */
  public MethodChannel(BinaryMessenger messenger, String name) {
    this(messenger, name, StandardMethodCodec.INSTANCE);
  }

  /**
   * Creates a new channel associated with the specified {@link BinaryMessenger} and with the
   * specified name and {@link MethodCodec}.
   *
   * @param messenger a {@link BinaryMessenger}.
   * @param name a channel name String.
   * @param codec a {@link MessageCodec}.
   */
  public MethodChannel(BinaryMessenger messenger, String name, MethodCodec codec) {
    if (BuildConfig.DEBUG) {
      if (messenger == null) {
        Log.e(TAG, "Parameter messenger must not be null.");
      }
      if (name == null) {
        Log.e(TAG, "Parameter name must not be null.");
      }
      if (codec == null) {
        Log.e(TAG, "Parameter codec must not be null.");
      }
    }
    this.messenger = messenger;
    this.name = name;
    this.codec = codec;
  }
  ...
}
```
创建 MethodChannel 的时候可以传入 codec， 默认为 StandardMethodCodec，StandardMethodCodec 分析 --> [StandardMethodCodec](#StandardMethodCodec_Java)   
我们使用最多的就是 MethodChannel 

##### EventChannel
```
public final class EventChannel {
  private static final String TAG = "EventChannel#";

  private final BinaryMessenger messenger;
  private final String name;
  private final MethodCodec codec;

  /**
   * Creates a new channel associated with the specified {@link BinaryMessenger} and with the
   * specified name and the standard {@link MethodCodec}.
   *
   * @param messenger a {@link BinaryMessenger}.
   * @param name a channel name String.
   */
  public EventChannel(BinaryMessenger messenger, String name) {
    this(messenger, name, StandardMethodCodec.INSTANCE);
  }

  /**
   * Creates a new channel associated with the specified {@link BinaryMessenger} and with the
   * specified name and {@link MethodCodec}.
   *
   * @param messenger a {@link BinaryMessenger}.
   * @param name a channel name String.
   * @param codec a {@link MessageCodec}.
   */
  public EventChannel(BinaryMessenger messenger, String name, MethodCodec codec) {
    if (BuildConfig.DEBUG) {
      if (messenger == null) {
        Log.e(TAG, "Parameter messenger must not be null.");
      }
      if (name == null) {
        Log.e(TAG, "Parameter name must not be null.");
      }
      if (codec == null) {
        Log.e(TAG, "Parameter codec must not be null.");
      }
    }
    this.messenger = messenger;
    this.name = name;
    this.codec = codec;
  }
  ...
}
```

创建 EventChannel 的时候可以传入 codec， 默认为 StandardMethodCodec，StandardMethodCodec 分析 --> [StandardMethodCodec](#StandardMethodCodec_Java)    
我们在部分场景下会使用到 EventChannel

#### 2.2.2 <span id="StandardMethodCodec_Java">StandardMethodCodec</span>
##### MessageCodec
所有编码器的父类是 MessageCodec，它是一个抽象类，里面定义了两个方法。编码和解码
```
public interface MessageCodec<T> {
  /**
   * Encodes the specified message into binary.
   *
   * @param message the T message, possibly null.
   * @return a ByteBuffer containing the encoding between position 0 and the current position, or
   *     null, if message is null.
   */
  @Nullable
  ByteBuffer encodeMessage(@Nullable T message);

  /**
   * Decodes the specified message from binary.
   *
   * @param message the {@link ByteBuffer} message, possibly null.
   * @return a T value representation of the bytes between the given buffer's current position and
   *     its limit, or null, if message is null.
   */
  @Nullable
  T decodeMessage(@Nullable ByteBuffer message);
}
```
##### StandardMessageCodec 
```
public class StandardMessageCodec implements MessageCodec<Object> {
  private static final String TAG = "StandardMessageCodec#";
  public static final StandardMessageCodec INSTANCE = new StandardMessageCodec();

  @Override
  public ByteBuffer encodeMessage(Object message) {
    if (message == null) {
      return null;
    }
    final ExposedByteArrayOutputStream stream = new ExposedByteArrayOutputStream();
    writeValue(stream, message);
    final ByteBuffer buffer = ByteBuffer.allocateDirect(stream.size());
    buffer.put(stream.buffer(), 0, stream.size());
    return buffer;
  }

  @Override
  public Object decodeMessage(ByteBuffer message) {
    if (message == null) {
      return null;
    }
    message.order(ByteOrder.nativeOrder());
    final Object value = readValue(message);
    if (message.hasRemaining()) {
      throw new IllegalArgumentException("Message corrupted");
    }
    return value;
  }

  private static final boolean LITTLE_ENDIAN = ByteOrder.nativeOrder() == ByteOrder.LITTLE_ENDIAN;
  private static final Charset UTF8 = Charset.forName("UTF8");
  private static final byte NULL = 0;
  private static final byte TRUE = 1;
  private static final byte FALSE = 2;
  private static final byte INT = 3;
  private static final byte LONG = 4;
  private static final byte BIGINT = 5;
  private static final byte DOUBLE = 6;
  private static final byte STRING = 7;
  private static final byte BYTE_ARRAY = 8;
  private static final byte INT_ARRAY = 9;
  private static final byte LONG_ARRAY = 10;
  private static final byte DOUBLE_ARRAY = 11;
  private static final byte LIST = 12;
  private static final byte MAP = 13;
```
首先定义了13种支持的数据类型，用int值表示。然后分别实现了编码和解码的方法
##### encodeMessage
```
@Override
public ByteBuffer encodeMessage(Object message) {
  if (message == null) {
    return null;
  }
  final ExposedByteArrayOutputStream stream = new ExposedByteArrayOutputStream();
  writeValue(stream, message);
  final ByteBuffer buffer = ByteBuffer.allocateDirect(stream.size());
  buffer.put(stream.buffer(), 0, stream.size());
  return buffer;
}
```
```
/**
 * Writes a type discriminator byte and then a byte serialization of the specified value to the
 * specified stream.
 *
 * <p>Subclasses can extend the codec by overriding this method, calling super for values that the
 * extension does not handle.
 */
protected void writeValue(ByteArrayOutputStream stream, Object value) {
  if (value == null || value.equals(null)) {
    stream.write(NULL);
  } else if (value instanceof Boolean) {
    stream.write(((Boolean) value).booleanValue() ? TRUE : FALSE);
  } else if (value instanceof Number) {
    if (value instanceof Integer || value instanceof Short || value instanceof Byte) {
      stream.write(INT);
      writeInt(stream, ((Number) value).intValue());
    } else if (value instanceof Long) {
      stream.write(LONG);
      writeLong(stream, (long) value);
    } else if (value instanceof Float || value instanceof Double) {
      stream.write(DOUBLE);
      writeAlignment(stream, 8);
      writeDouble(stream, ((Number) value).doubleValue());
    } else if (value instanceof BigInteger) {
      stream.write(BIGINT);
      writeBytes(stream, ((BigInteger) value).toString(16).getBytes(UTF8));
    } else {
      throw new IllegalArgumentException("Unsupported Number type: " + value.getClass());
    }
  } else if (value instanceof String) {
    stream.write(STRING);
    writeBytes(stream, ((String) value).getBytes(UTF8));
  } else if (value instanceof byte[]) {
    stream.write(BYTE_ARRAY);
    writeBytes(stream, (byte[]) value);
  } else if (value instanceof int[]) {
    stream.write(INT_ARRAY);
    final int[] array = (int[]) value;
    writeSize(stream, array.length);
    writeAlignment(stream, 4);
    for (final int n : array) {
      writeInt(stream, n);
    }
  } else if (value instanceof long[]) {
    stream.write(LONG_ARRAY);
    final long[] array = (long[]) value;
    writeSize(stream, array.length);
    writeAlignment(stream, 8);
    for (final long n : array) {
      writeLong(stream, n);
    }
  } else if (value instanceof double[]) {
    stream.write(DOUBLE_ARRAY);
    final double[] array = (double[]) value;
    writeSize(stream, array.length);
    writeAlignment(stream, 8);
    for (final double d : array) {
      writeDouble(stream, d);
    }
  } else if (value instanceof List) {
    stream.write(LIST);
    final List<?> list = (List) value;
    writeSize(stream, list.size());
    for (final Object o : list) {
      writeValue(stream, o);
    }
  } else if (value instanceof Map) {
    stream.write(MAP);
    final Map<?, ?> map = (Map) value;
    writeSize(stream, map.size());
    for (final Entry<?, ?> entry : map.entrySet()) {
      writeValue(stream, entry.getKey());
      writeValue(stream, entry.getValue());
    }
  } else {
    throw new IllegalArgumentException("Unsupported value: " + value);
  }
}
```
编码过程
1. 首先定义一个字节缓冲区，用于存放数据编码成的二进制数据
2. 判断传入的对象的类型，是否在定义的13种数据类型之内，如果不在的话抛出` IllegalArgumentException("Unsupported value: " + value)`异常
3. 开始对数据进行编码。  
这些数据分为两种，  
一种是固定数据类型，这种数据长度固定。另一种是不固定数据类型，这种数据长度不固定，需要再次编码或者需要附加其他参数进行编码。  

### 2.3 编码详细过程

#### 2.3.1 null、true， false
这三种类型直接在缓冲区里写入对应的类型标识，分别为0、1、2
##### Dart
```
if (value == null) {
  buffer.putUint8(_valueNull);
} else if (value is bool) {
  buffer.putUint8(value ? _valueTrue : _valueFalse);
}
```
##### Java
```
if (value == null || value.equals(null)) {
  stream.write(NULL);
} else if (value instanceof Boolean) {
  stream.write(((Boolean) value).booleanValue() ? TRUE : FALSE);
} 
```
#### 2.3.2 整型和浮点型数据
Dart中的`int`分为整形和长整型，编码的时候先写入类型标识，再写入数据，类型标识分别为3、4。浮点型标识为6。编码阶段没有`larger integers`类型
```
if (value == null) {
  buffer.putUint8(_valueNull);
} else if (value is bool) {
  buffer.putUint8(value ? _valueTrue : _valueFalse);
} else if (value is double) {  // Double precedes int because in JS everything is a double.
                               // Therefore in JS, both `is int` and `is double` always
                               // return `true`. If we check int first, we'll end up treating
                               // all numbers as ints and attempt the int32/int64 conversion,
                               // which is wrong. This precedence rule is irrelevant when
                               // decoding because we use tags to detect the type of value.
  buffer.putUint8(_valueFloat64);
  buffer.putFloat64(value);
} else if (value is int) {
  if (-0x7fffffff - 1 <= value && value <= 0x7fffffff) {
    buffer.putUint8(_valueInt32);
    buffer.putInt32(value);
  } else {
    buffer.putUint8(_valueInt64);
    buffer.putInt64(value);
  }
} 
```
Java中的`Number`编码的时候先写入类型标识，再写入数据。整形类型标识分别为3、4，BigInteger类型标识为5，浮点型标识为6。其中BigInteger按照String类型编码再转byte数组传递，byte和short转int再处理
```
if (value instanceof Number) {
  if (value instanceof Integer || value instanceof Short || value instanceof Byte) {
    stream.write(INT);
    writeInt(stream, ((Number) value).intValue());
  } else if (value instanceof Long) {
    stream.write(LONG);
    writeLong(stream, (long) value);
  } else if (value instanceof Float || value instanceof Double) {
    stream.write(DOUBLE);
    writeAlignment(stream, 8);
    writeDouble(stream, ((Number) value).doubleValue());
  } else if (value instanceof BigInteger) {
    stream.write(BIGINT);
    writeBytes(stream, ((BigInteger) value).toString(16).getBytes(UTF8));
  } else {
    throw new IllegalArgumentException("Unsupported Number type: " + value.getClass());
  }
}
```
这个过程中Double类型的数据编码的时候需要一次对齐操作。这是因为在C层读取数据的时候直接操作的内存地址，基于二进制的关系，所以需要对齐到相应的所需byte数量为一个单位的维度

#### 2.3.3 字符串类型和byte数组
Dart
```
if (value is String) {
  buffer.putUint8(_valueString);
  final Uint8List bytes = utf8.encoder.convert(value);
  writeSize(buffer, bytes.length);
  buffer.putUint8List(bytes);
} else if (value is Uint8List) {
  buffer.putUint8(_valueUint8List);
  writeSize(buffer, value.length);
  buffer.putUint8List(value);
} 
```
Java
```
if (value instanceof String) {
  stream.write(STRING);
  writeBytes(stream, ((String) value).getBytes(UTF8));
} else if (value instanceof byte[]) {
  stream.write(BYTE_ARRAY);
  writeBytes(stream, (byte[]) value);
} 
```
```
/** Writes the length and then the actual bytes of the specified array to the specified stream. */
protected static final void writeBytes(ByteArrayOutputStream stream, byte[] bytes) {
  writeSize(stream, bytes.length);
  stream.write(bytes, 0, bytes.length);
}
```
这两种类型标识分别为7、8。编码的时候都会按照byte数组处理，先写入类型标识，再写入byte数组长度，最后写入数组

#### 2.3.5 数组类型，包括整形、长整形、浮点型数据
Dart
```
if (value is Uint8List) {
  buffer.putUint8(_valueUint8List);
  writeSize(buffer, value.length);
  buffer.putUint8List(value);
} else if (value is Int32List) {
  buffer.putUint8(_valueInt32List);
  writeSize(buffer, value.length);
  buffer.putInt32List(value);
} else if (value is Int64List) {
  buffer.putUint8(_valueInt64List);
  writeSize(buffer, value.length);
  buffer.putInt64List(value);
} else if (value is Float64List) {
  buffer.putUint8(_valueFloat64List);
  writeSize(buffer, value.length);
  buffer.putFloat64List(value);
}
```
Java
```
if (value instanceof int[]) {
  stream.write(INT_ARRAY);
  final int[] array = (int[]) value;
  writeSize(stream, array.length);
  writeAlignment(stream, 4);
  for (final int n : array) {
    writeInt(stream, n);
  }
} else if (value instanceof long[]) {
  stream.write(LONG_ARRAY);
  final long[] array = (long[]) value;
  writeSize(stream, array.length);
  writeAlignment(stream, 8);
  for (final long n : array) {
    writeLong(stream, n);
  }
} else if (value instanceof double[]) {
  stream.write(DOUBLE_ARRAY);
  final double[] array = (double[]) value;
  writeSize(stream, array.length);
  writeAlignment(stream, 8);
  for (final double d : array) {
    writeDouble(stream, d);
  }
} 
```
这三种类型标识分别为9、10、11。编码的时候都会先写入类型标识，再写入数组长度，然后`对齐缓冲区`最后写入数组
#### 2.3.6 列表(List)型数据
Dart
```
if (value is List) {
  buffer.putUint8(_valueList);
  writeSize(buffer, value.length);
  for (final Object? item in value) {
    writeValue(buffer, item);
  }
} 
```
Java
```
if (value instanceof List) {
  stream.write(LIST);
  final List<?> list = (List) value;
  writeSize(stream, list.size());
  for (final Object o : list) {
    writeValue(stream, o);
  }
}
```
这种类型标识为12。编码的时候都会先写入类型标识，再写入列表长度，然后递归调用`writeValue`方法，分别写入列表中的每一个对象。
#### 2.3.7 字典(Map)型数据
Dart
```
if (value is Map) {
  buffer.putUint8(_valueMap);
  writeSize(buffer, value.length);
  value.forEach((Object? key, Object? value) {
    writeValue(buffer, key);
    writeValue(buffer, value);
  });
} 
```
Java
```
if (value instanceof Map) {
  stream.write(MAP);
  final Map<?, ?> map = (Map) value;
  writeSize(stream, map.size());
  for (final Entry<?, ?> entry : map.entrySet()) {
    writeValue(stream, entry.getKey());
    writeValue(stream, entry.getValue());
  }
} 
```
这种类型标识为13。编码的时候都会先写入类型标识，再写入字典长度，然后递归调用`writeValue`方法，分别写入字典中的每一个key，在写入value

### 2.4 解码详细过程

解码过程作为编码过程的逆运算，相对比较简单。
Dart
```
Object? readValue(ReadBuffer buffer) {
  if (!buffer.hasRemaining)
    throw const FormatException('Message corrupted');
  final int type = buffer.getUint8();
  return readValueOfType(type, buffer);
}
```
```
Object? readValueOfType(int type, ReadBuffer buffer) {
  switch (type) {
    case _valueNull:
      return null;
    case _valueTrue:
      return true;
    case _valueFalse:
      return false;
    case _valueInt32:
      return buffer.getInt32();
    case _valueInt64:
      return buffer.getInt64();
    case _valueFloat64:
      return buffer.getFloat64();
    case _valueLargeInt:
    case _valueString:
      final int length = readSize(buffer);
      return utf8.decoder.convert(buffer.getUint8List(length));
    case _valueUint8List:
      final int length = readSize(buffer);
      return buffer.getUint8List(length);
    case _valueInt32List:
      final int length = readSize(buffer);
      return buffer.getInt32List(length);
    case _valueInt64List:
      final int length = readSize(buffer);
      return buffer.getInt64List(length);
    case _valueFloat64List:
      final int length = readSize(buffer);
      return buffer.getFloat64List(length);
    case _valueList:
      final int length = readSize(buffer);
      final List<Object?> result = List<Object?>.filled(length, null, growable: false);
      for (int i = 0; i < length; i++)
        result[i] = readValue(buffer);
      return result;
    case _valueMap:
      final int length = readSize(buffer);
      final Map<Object?, Object?> result = <Object?, Object?>{};
      for (int i = 0; i < length; i++)
        result[readValue(buffer)] = readValue(buffer);
      return result;
    default: throw const FormatException('Message corrupted');
  }
}
```
Java

```
protected final Object readValue(ByteBuffer buffer) {
  if (!buffer.hasRemaining()) {
    throw new IllegalArgumentException("Message corrupted");
  }
  final byte type = buffer.get();
  return readValueOfType(type, buffer);
}
```
```
protected Object readValueOfType(byte type, ByteBuffer buffer) {
  final Object result;
  switch (type) {
    case NULL:
      result = null;
      break;
    case TRUE:
      result = true;
      break;
    case FALSE:
      result = false;
      break;
    case INT:
      result = buffer.getInt();
      break;
    case LONG:
      result = buffer.getLong();
      break;
    case BIGINT:
      {
        final byte[] hex = readBytes(buffer);
        result = new BigInteger(new String(hex, UTF8), 16);
        break;
      }
    case DOUBLE:
      readAlignment(buffer, 8);
      result = buffer.getDouble();
      break;
    case STRING:
      {
        final byte[] bytes = readBytes(buffer);
        result = new String(bytes, UTF8);
        break;
      }
    case BYTE_ARRAY:
      {
        result = readBytes(buffer);
        break;
      }
    case INT_ARRAY:
      {
        final int length = readSize(buffer);
        final int[] array = new int[length];
        readAlignment(buffer, 4);
        buffer.asIntBuffer().get(array);
        result = array;
        buffer.position(buffer.position() + 4 * length);
        break;
      }
    case LONG_ARRAY:
      {
        final int length = readSize(buffer);
        final long[] array = new long[length];
        readAlignment(buffer, 8);
        buffer.asLongBuffer().get(array);
        result = array;
        buffer.position(buffer.position() + 8 * length);
        break;
      }
    case DOUBLE_ARRAY:
      {
        final int length = readSize(buffer);
        final double[] array = new double[length];
        readAlignment(buffer, 8);
        buffer.asDoubleBuffer().get(array);
        result = array;
        buffer.position(buffer.position() + 8 * length);
        break;
      }
    case LIST:
      {
        final int size = readSize(buffer);
        final List<Object> list = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
          list.add(readValue(buffer));
        }
        result = list;
        break;
      }
    case MAP:
      {
        final int size = readSize(buffer);
        final Map<Object, Object> map = new HashMap<>();
        for (int i = 0; i < size; i++) {
          map.put(readValue(buffer), readValue(buffer));
        }
        result = map;
        break;
      }
    default:
      throw new IllegalArgumentException("Message corrupted");
  }
  return result;
}
```
首先判断缓冲区是否还有剩余数据，在读的过程中，每种数据都是按照特定长度编码的，解析结束的时候应该正好到末尾，否则就证明编码有问题或者数据传输有问题，直接抛出来异常。  
然后读取一位类型标识，
1. 如果是null、true、false、整形，则直接读取相应类型数据  
2. 如果是浮点型，则需要对齐缓冲区在读取相应类型数据  
3. 如果是数组类型（整形、长整形、浮点型数据），则需要读取数组长度、对齐缓冲区在读取相应类型
4. 如果是byte数组类型或者String类型，则需要读取byte数组长度再去读byte数组，String类型按照UTF8进行编码
5. 如果是列表(List)类型，则需要读取列表长度，创建列表，按照列表长度递归读取每一条数据
6. 如果是字典(Map)类型，则需要读取字典长度，创建字典，按照字典长度递归读取key和value存放到字典中  

至此，解码的过程就完成了

## 3. 小结
编码过程总结起来就是在字节缓冲区，先写入一个byte作为类型标识、再写入数据。  
但是在写入部分类型的数据的时候发现长度不固定（数组），则会再借助一个byte记录数据长度。  
遇到一些数据对象类型为复杂类型（List、Map），则写入长度后再对每一个对象进行再次编码。

在这其中有些数据受限于底层内存结构和API的原因，还需要进行缓冲区对齐。
