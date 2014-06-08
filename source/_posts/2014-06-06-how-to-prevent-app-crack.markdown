---
layout: post
title: "如何防止客户端被破解"
date: 2014-06-06 13:59
comments: true
categories: iOS
---

很多应用都需要用户登录或者签名认证，这可能需要在客户端保存登录信息、签名密钥、加密算法等。如何保证这些重要信息不被窃取，算法不被破解，这些成为应用开发中很重要的内容，同样也是最容易忽视的地方。一个小小的细节可能就成为整个系统的突破口，这里从实际工程角度总结了一些容易忽视的细节和常用的方法。

<!--more-->

### 密钥保存在外部

  - Keychain
    
    密钥保存在Keychain并不安全，iOS越狱后可以导出Keychain的内容。应该尽量避免存放重要信息(如：token、用户名、密码等)在Keychain中，即使要存放，也一定要加密后存放。参考<http://blog.csdn.net/yiyaaixuexi/article/details/18404343>

  - 文件

    保存在app bundle、plist等配置文件更不安全，但可以使用隐写术等方式迷惑hackers。有请Lena示范：

    {% img /images/Lena.jpg %}    {% img /images/Lena-secret.jpg %}

    两张图片看起来是一模一样的，但是右边的图片里却夹带了一些其他内容，这就是潜伏在Lena中的密码，用diff工具比较下这两张图片，你会发现不同的地方是右边的图片最后附加了一串字符：`app secret is "abcdefg123456"`。 这里的隐写方式很简单：`cat file >> Lena.jpg`，既不破坏图片原本的信息(或者损失一点点原有信息)，又能附加额外的信息，这就是隐写术的原理。这里只是一个简单的例子，没有人真这么使用。有很多更隐蔽的做法，比如把要隐藏的信息分散到图片的每个像素中，例如RGB888的图片，对红蓝分量最后一个bit位进行修改并不会影响图片的质量(因为人眼对对红蓝不敏感)，这样一个像素(3byte)就可以存储2bit的信息，4个像素(12byte)就可以夹带1byte的信息了。

    > Xcode打包时会对png图片做特殊处理，如果将密码携带在png中，可能会在使用的时候无法复原。当然现在的隐写术非常多，不只是图片能作为载体，视频、音乐等文件都可以，隐写的方法也多种多样，选择适合自己的就行，据说基地组织就是通过岛国电影传递信息的。

### 写在代码里安全吗？
下面的代码很常见

{% codeblock lang:objc %}
  #define kSecret "abcd1234"

  // 或者
  const char* kSecret = "abcd1234";
{% endcodeblock %}

这是非常危险的，因为常量会被直接编译到可执行文件的data段，只要对生成的可执行文件使用`strings`、`otool`等命令就可以dump出原始字符串。

### 对密码加密
为了使密码不直接出现在可执行文件中，可以对密码加密存储，使用的时候再解密。
例如用AES对密码`abcd1234`加密，对称密钥为`kCipherKey="abcdefgh12345678"`，加密后的密码用`kSecret`表示。使用密码时，再通过`kCipherKey`和`kSecret`计算出来：

{% codeblock lang:objc snippet1 %}
const char* kCipherKey="abcdefgh12345678";
const char* kSecret="\x7e\x77\x64\x3c\xa7\xd4\x6d\x46\x29\x8b\xe3\x23\x9f\x1a\x5c\xdb";

char* getSecret() {
    char* buf = NULL;
    CCCryptorRef cryptor = NULL;
    uint8_t iv[kCCBlockSizeAES128];
    memset(iv, 0, kCCBlockSizeAES128);
    
    size_t bufsize = 0;
    size_t moved = 0;
    size_t total = 0;
    
    size_t inLength = strlen(kSecret);
    
    CCCryptorCreate(kCCDecrypt, kCCAlgorithmAES128,
                    kCCOptionPKCS7Padding,
                    kCipherKey, strlen(kCipherKey),
                    iv, &cryptor);
    bufsize = CCCryptorGetOutputLength(cryptor, inLength, true);
    buf = (char*)malloc(bufsize);
    memset(buf, 0, bufsize);
    
    CCCryptorUpdate(cryptor,
                    kSecret,inLength,
                    buf, bufsize, &moved);
    total += moved;
    
    CCCryptorFinal(cryptor,
                   buf+total,
                   bufsize-total, &moved);
    CCCryptorRelease(cryptor);
    
    return buf;
}
 

int main(int argc, char * argv[]) {
  char* secret = getSecret();
  printf("%s\n", secret);
  free(secret);
}

{% endcodeblock %}

上面的代码不再明文出现`abcd1234`，而是被加密算子`kCipherKey`和加密后的密钥`kSecret`替代，密码只是在需要的时候临时计算出来。但是这里仍然有缺陷：加密算子`kCipherKey`和加密后的密钥`kSecret`仍然存储在可执行文件的data段中，留下了蛛丝马迹。我们可以给`kCipherKey`取一个有迷惑性的字符串，比如`"network error, timeout"`或者使用非字符值，使其不可读。但这都不完美，不在data段中存储这些信息最好。

### 参数传递的秘密
上面的代码稍做修改

{% codeblock lang:objc snippet2 %}

// 注意这里
#define kCipherKey ((uint8_t[]){'a','b','c','d','e','f','g','h','1','2','3','4','5','6','7','8'})
#define kSecret ((uint8_t[]){0x7e,0x77,0x64,0x3c,0xa7,0xd4,0x6d,0x46,0x29,0x8b,0xe3,0x23,0x9f,0x1a,0x5c,0xdb})

char* getSecret() {
    char* buf = NULL;
    CCCryptorRef cryptor = NULL;
    uint8_t iv[kCCBlockSizeAES128];
    memset(iv, 0, kCCBlockSizeAES128);
    
    size_t bufsize = 0;
    size_t moved = 0;
    size_t total = 0;
    
    size_t inLength = sizeof(kSecret);
    
    CCCryptorCreate(kCCDecrypt, kCCAlgorithmAES128,
                    kCCOptionPKCS7Padding,
                    kCipherKey, sizeof(kCipherKey),
                    iv, &cryptor);
    bufsize = CCCryptorGetOutputLength(cryptor, inLength, true);
    buf = (char*)malloc(bufsize);
    memset(buf, 0, bufsize);
    
    CCCryptorUpdate(cryptor,
                    kSecret,inLength,
                    buf, bufsize, &moved);
    total += moved;
    
    CCCryptorFinal(cryptor,
                   buf+total,
                   bufsize-total, &moved);
    CCCryptorRelease(cryptor);
    
    return buf;
}

int main(int argc, char * argv[]) {
    char* secret = getSecret();
    printf("%s\n", secret);
    free(secret);
}
{% endcodeblock %}

看似和上面代码没什么区别，除了传入的参数类型变了，其余没什么变化。正是这一点带来了巨大的变化，对比一下调用CCCryptorCreate时的汇编代码：

{% codeblock lang:objc snippet1-disassemble%}
Demo`getSecret at main.m:58:
0x31f04:  push   {r4, r5, r6, r7, lr}
0x31f06:  add    r7, sp, #0xc
0x31f08:  push.w {r8, r10, r11}
0x31f0c:  sub    sp, #0x28
0x31f0e:  movw   r0, #0x112a
0x31f12:  vmov.i32 q8, #0x0
0x31f16:  movt   r0, #0x0
0x31f1a:  movw   r8, #0x16d8
0x31f1e:  add    r0, pc
0x31f20:  movt   r8, #0x0
0x31f24:  add    r8, pc
0x31f26:  add    r6, sp, #0x14
0x31f28:  ldr.w  r10, [r0]
0x31f2c:  ldr.w  r0, [r10]
0x31f30:  str    r0, [sp, #0x24]
0x31f32:  movs   r0, #0x0
0x31f34:  str    r0, [sp, #0x10]
0x31f36:  str    r0, [sp, #0xc]
0x31f38:  ldr.w  r0, [r8]
0x31f3c:  vst1.32 {d16, d17}, [r6]
0x31f40:  blx    0x32ffc                   ; symbol stub for: strlen
0x31f44:  mov    r4, r0
0x31f46:  movw   r0, #0x16aa
0x31f4a:  movt   r0, #0x0
0x31f4e:  add    r0, pc
0x31f50:  ldr    r5, [r0]
0x31f52:  mov    r0, r5
0x31f54:  blx    0x32ffc                   ; symbol stub for: strlen
0x31f58:  add    r1, sp, #0x10
0x31f5a:  stm.w  sp, {r0, r6}
0x31f5e:  movs   r0, #0x1
0x31f60:  str    r1, [sp, #0x8]
0x31f62:  movs   r1, #0x0
0x31f64:  movs   r2, #0x1
0x31f66:  mov    r3, r5
0x31f68:  blx    0x32fd4                   ; symbol stub for: CCCryptorCreate
{% endcodeblock %}

{% codeblock lang:objc snippet2-disassemble %}
Demo`getSecret at main.m:23:
0x4de84:  push   {r4, r5, r6, r7, lr}
0x4de86:  add    r7, sp, #0xc
0x4de88:  push.w {r8, r10, r11}
0x4de8c:  sub    sp, #0x3c
0x4de8e:  movw   r0, #0x11a6
0x4de92:  movs   r1, #0x0
0x4de94:  movt   r0, #0x0
0x4de98:  movs   r6, #0x64
0x4de9a:  add    r0, pc
0x4de9c:  vmov.i32 q8, #0x0
0x4dea0:  ldr    r5, [r0]
0x4dea2:  ldr    r0, [r5]
0x4dea4:  str    r0, [r7, #-28]
0x4dea8:  sub.w  r0, r7, #0x2c
0x4deac:  str    r1, [r7, #-80]
0x4deb0:  str    r1, [r7, #-84]
0x4deb4:  movs   r1, #0x61
0x4deb6:  strb   r1, [r7, #-60]
0x4deba:  movs   r1, #0x62
0x4debc:  strb   r1, [r7, #-59]
0x4dec0:  movs   r1, #0x63
0x4dec2:  strb   r1, [r7, #-58]
0x4dec6:  movs   r1, #0x65
0x4dec8:  strb   r6, [r7, #-57]
0x4decc:  strb   r1, [r7, #-56]
0x4ded0:  movs   r1, #0x66
0x4ded2:  strb   r1, [r7, #-55]
0x4ded6:  movs   r1, #0x67
0x4ded8:  strb   r1, [r7, #-54]
0x4dedc:  movs   r1, #0x68
0x4dede:  strb   r1, [r7, #-53]
0x4dee2:  movs   r1, #0x31
0x4dee4:  strb   r1, [r7, #-52]
0x4dee8:  movs   r1, #0x32
0x4deea:  strb   r1, [r7, #-51]
0x4deee:  movs   r1, #0x33
0x4def0:  strb   r1, [r7, #-50]
0x4def4:  movs   r1, #0x34
0x4def6:  strb   r1, [r7, #-49]
0x4defa:  movs   r1, #0x35
0x4defc:  strb   r1, [r7, #-48]
0x4df00:  movs   r1, #0x36
0x4df02:  strb   r1, [r7, #-47]
0x4df06:  movs   r1, #0x37
0x4df08:  strb   r1, [r7, #-46]
0x4df0c:  movs   r1, #0x38
0x4df0e:  vst1.32 {d16, d17}, [r0]
0x4df12:  strb   r1, [r7, #-45]
0x4df16:  sub    sp, #0xc
0x4df18:  movs   r2, #0x10
0x4df1a:  sub.w  r1, r7, #0x50
0x4df1e:  sub.w  r3, r7, #0x3c
0x4df22:  str    r2, [sp]
0x4df24:  str    r0, [sp, #0x4]
0x4df26:  movs   r0, #0x1
0x4df28:  str    r1, [sp, #0x8]
0x4df2a:  movs   r1, #0x0
0x4df2c:  movs   r2, #0x1
0x4df2e:  blx    0x4efdc                   ; symbol stub for: CCCryptorCreate
{% endcodeblock %}

注意CCCryptorCreate的第四个参数，对应寄存器`r3`，第一段代码的`r3`的值是从text段直接获取，因为这只是data段的相对地址，编译时就确定了。而再看第二段代码，出现了大量的`strb`指令，分析知这段指令是把`abcdefgh12345678`每个字符逐个压进执行栈的连续地址中，然后`r3`取相应的连续地址的首地址。也就是说`kCipherKey`不再直接存储在data段，而是打散到多个指令中，成为指令的一部分(指令在text段)，当代码运行时，这些指令再把`kCipherKey`原始内容逐个压入执行栈中构成字符串，然后用栈中字符串首地址作为参数传给`CCCryptorCreate`，这使得每次调用时传入的字符串地址都不同。函数`CCCryptorUpdate`原理也是一样。函数`getSecret()`执行完之后，他的执行栈被清空，`kCipherKey`和`kSecret`原始信息也一起从栈中清楚，这样重要信息不会常驻内存，只是用到时才进入内存，用完立即清除，这可以有效预防内存扫描器。

> 上面的代码仍然不够完美，首先`getSecret`是函数形式、而且密码通过返回值传递，容易被分析破解；其次返回的密码的buffer内存需要调用者释放，代码不够整洁，而且调用者容易忘记。

### 宏改造
{% codeblock lang:objc snippet3 %}
#define kCipherKey ((uint8_t[]){'a','b','c','d','e','f','g','h','1','2','3','4','5','6','7','8'})
#define kSecret ((uint8_t[]){0x7e,0x77,0x64,0x3c,0xa7,0xd4,0x6d,0x46,0x29,0x8b,0xe3,0x23,0x9f,0x1a,0x5c,0xdb})

#define kAppSecret                                          \
({                                                          \
    size_t outLength = 0;                                   \
    char* buf = getSecret(outLength);                      \
    [[NSString alloc] initWithBytes:buf                     \
                            length:outLength                \
                          encoding:NSASCIIStringEncoding];  \
})

#define _CHK_CCSUCC(status, outLength)                      \
    if ((status) != kCCSuccess) {                           \
    outLength = 0;                                          \
    goto end;                                               \
}

#define getSecret(outLength)                                \
({                                                          \
    __label__ end;                                          \
    char* buf = NULL;                                       \
                                                            \
    CCCryptorRef cryptor = NULL;                            \
    uint8_t iv[kCCBlockSizeAES128];                         \
    memset(iv, 0, kCCBlockSizeAES128);                      \
                                                            \
    size_t bufsize = 0;                                     \
    size_t moved = 0;                                       \
    size_t total = 0;                                       \
    size_t inLength = sizeof(kSecret);                      \
                                                            \
    _CHK_CCSUCC(CCCryptorCreate(kCCDecrypt,                 \
                kCCAlgorithmAES128,                         \
                kCCOptionPKCS7Padding,                      \
                kCipherKey, sizeof(kCipherKey),             \
                iv, &cryptor), outLength);                  \
    bufsize = CCCryptorGetOutputLength(cryptor,             \
                                       inLength, true);     \
    buf = (char*)alloca(bufsize);                           \
    memset(buf, 0, bufsize);                                \
                                                            \
    _CHK_CCSUCC(CCCryptorUpdate(cryptor,                    \
                                kSecret,inLength,           \
                                buf, bufsize, &moved),      \
                                outLength);                 \
    total += moved;                                         \
                                                            \
    _CHK_CCSUCC(CCCryptorFinal(cryptor,                     \
                               buf+total,                   \
                               bufsize-total, &moved),      \
                               outLength);                  \
    total += moved;                                         \
                                                            \
    outLength = total;                                      \
end:                                                        \
    if (cryptor) {                                          \
        CCCryptorRelease(cryptor);                          \
    }                                                       \
    buf;                                                    \
})

int main(int argc, char * argv[]) {
    NSLog(@"%@",kAppSecret);
}
{% endcodeblock %}

这段代码稍微改造了一下，加入了一些必要的检测，让调用者更加简单，宏`kAppSecret`将密码包装成NSString对象。更重要的是，`buf`的内存不再是`malloc`到堆上，而是`alloca`到栈上(或者使用C99的变长数组)，确切的说是*调用者*的栈，*调用者*不再需要手动释放内存；另外，因为`kAppSecret`是宏，没有有明确的入口，静态分析更加困难。

> 
__这里用了宏定义的两个技巧：__

  - 带返回值的宏

{% codeblock lang:objc %}
#define SOME_MACRO \
({                 \
  expression;      \
})                 \
{% endcodeblock %}
最后一个表达式的值就是宏的返回值，使用时更像函数的返回值。

  - 局部标签

局部标签用`__label__`定义。如果标签`end`没有`__label__`修饰，在同一个函数中多次使用`kAppSecret`将产生编译错误，因为宏展开后相当于定义了多个`end`标签，标签重复定义。

### 函数指针
在客户端访问Web Server的时候，Server往往要验证请求是否来自合法的客户端，而不是攻击者伪造的请求，这就需要客户端签名。例如OAuth的签名算法。如果自己定义签名算法，不希望别人知道签名的过程，就需要保护算法不被破解。例如签名算法是`HMAC-SHA1(key,MD5(data))`：

{% codeblock lang:objc signature1 %}
- (NSString*) signatureData:(NSString*)data byKey:(NSString*)key {
    unsigned char md[16];
    CC_MD5(data.UTF8String, (CC_LONG)data.length, md);
    
    char mac[CC_SHA1_DIGEST_LENGTH];
    CCHmac(kCCHmacAlgSHA1, key.UTF8String, key.length, md, 16, mac);
    return [self hexStringFromBytes:mac length:CC_SHA1_DIGEST_LENGTH];
}
{% endcodeblock %}

这段代码本身没有问题，但是对系统函数的直接调用导致代码容易被静态分析，用IDA、otool等静态分析工具可以很容易的知道这个函数的workflow，签名过程被轻易破解。为了防备静态分析，可以使用函数指针间接调用函数：
{% codeblock lang:objc signature2 %}
@interface SecurityService : NSObject

- (id) initWithMD5Function:(void*)md5 HMACFunction:(void*)hmac;
- (NSString*) signatureData:(NSString*)data byKey:(NSString*)key;

@end

@implementation SecurityService {
    void*   _md5Funcation;
    void*   _hmacFuncation;
}

- (id) initWithMD5Function:(void*)md5 HMACFunction:(void*)hmac {
    if (self = [super init]) {
        _md5Funcation = (void*)(unsigned long)((uint)&_md5Funcation^(uint)md5);
        _hmacFuncation = (void*)(unsigned long)((uint)&_hmacFuncation^(uint)hmac);
        
    }
    return self;
}


- (NSString*) signatureData:(NSString*)data byKey:(NSString*)key {
    unsigned char md[16];
    void* func = (void*)(unsigned long)((uint)&_md5Funcation ^ (uint)_md5Funcation);
    ((unsigned char (*)(const void*, CC_LONG, unsigned char*))func)(data.UTF8String,
                                                                    (CC_LONG)data.length,
                                                                    md);
    
    char mac[CC_SHA1_DIGEST_LENGTH];
    func = (void*)(unsigned long)((uint)&_hmacFuncation ^ (uint)_hmacFuncation);
    ((void(*)(CCHmacAlgorithm, const void*, size_t, const void*, size_t, void*))func)(kCCHmacAlgSHA1,
                                                                                      key.UTF8String,
                                                                                      key.length,
                                                                                      md,
                                                                                      16,
                                                                                      mac);
    return [self hexStringFromBytes:mac length:CC_SHA1_DIGEST_LENGTH];
}

- (NSString*) hexStringFromBytes:(char*)bytes length:(NSUInteger)length {
    NSMutableString *hexStr=[[NSMutableString alloc] initWithCapacity:2*length];
    
    for(int i=0;i<length;i++) {
        [hexStr appendFormat:@"%02x", bytes[i]&0xff];
    }
    
    return [NSString stringWithString:hexStr];
}
@end

int main(int argc, char * argv[]) {
    SecurityService* ss = [[SecurityService alloc] initWithMD5Function:CC_MD5 HMACFunction:CCHmac];
    NSString* sign = [ss signatureData:@"1234" byKey:kAppSecret];
    NSLog(@"%@",sign);
}
{% endcodeblock %}

签名类初始化的时候，保存了HASH函数的地址值，执行签名的时，通过HASH函数的地址间接调用，这样静态分析工具分析到这里的时候，只能看到调用了某个地址，而不知道调用的具体函数，隐藏了真实目的。


> 这里不是直接将函数地址赋值给对象属性，而是用属性的地址与函数的地址做抑或运算。这样做主要有两个原因：
>>  
  - 直接赋值可能被编译器优化，编译器自动将使用该属性的地方替换成函数本身；
>>
  - 类实例的创建有随机性，属性的内存地址也具有随机性，用属性地址加密函数地址，这样属性值在每次运行时都不一样；

在Android或其他平台还可以用dlsym来获取函数地址：
{% codeblock lang:objc 伪代码 %}
char data[] = {0x32, 0x71, 0x0b, 0x48, 0xe3, 0xbc, 0x6a, 0x27, 0x8e, 0xca, 0x3b, 0x0e};
char sym[sizeof(data)/2 + 1] = {0};
for (int i=0; i<sizeof(data); i+=2) {
    sym[i/2] = data[i] ^ data[i+1];
}
// sym = "CC_MD5"
void* md5 = dlsym(handle, sym);
{% endcodeblock %}

### 代码混淆
因为objc代码的动态性，编译器会在binary中留下类名、函数名等信息，这些信息是可以被`class-dump-z`等工具提取的，友好的命名让程序猿更方便，但同时也方便了破解者。对安全相关的重要模块类，可以故意混淆类名，让人不容易轻易联想到该的真实目的。比如把类名`SecurityService`改为`FIFA`。一些重要模块可以使用C/C++语言实现，编译器对C/C++并不会保留类名、方法名等信息。

> 使用混淆的名字对使用者很不方便，例如`[[FIFA alloc] initWithMD5Function:CC_MD5 HMACFunction:CCHmac];`这样的代码让人不知其意。可以用宏定义一个友好的名字来替代原来的类名`#define SecurityService FIFA`

### 动态调试
除了静态分析，破解者还可以使用gdb动态调试、Theos hook来分析代码，常用的系统加密函数、HASH函数都可能成为监控的对象，只要监控传递给他们的参数、调用栈就能轻松分析出密钥、算法等。所以使用系统的加密函数虽然节省开发时间、执行效率高，但并不是很安全，有些算法可能需要自己重写。

### 反调试
可以用ptrace等方法阻止gdb注入，但ptrace本身也可以被静态修改或hook。只好从多方面考虑，尽量提高安全性，比如检查binary签名是否匹配；检查手机是否越狱，越机做特殊处理等。参考<http://blog.csdn.net/yiyaaixuexi/article/details/20286929>

### 代码加密
类似UPX等加壳技术在iOS中无法使用，因为iOS堆、栈内存都没有执行权限，这也是jit技术无法在iOS中使用的原因(除非苹果自己或越狱系统)。

### 脚本
将算法用脚本实现，脚本被编译成bytecode后，app解释执行bytecode指令，可以有效的防止动态调试，因为hackers看到的将是一条条的指令在switch case中执行，就像把图片的像素逐个地放给别人看，当他看完全部的像素后也不一定知道整张图片是什么样子。当然用脚本方式会增大开发成本，对执行效率也有一定影响，需要开发者在安全、开发成本、性能三者之间找个平衡点。

### 最后
软件保护技术多种多样，甚至有硬件级的加密模块TPM(Trusted Platform Module)。总之没有绝对的安全，但危险显然也只是相对的，只要提高编码意识，注意防护就可以把风险降到最低。

