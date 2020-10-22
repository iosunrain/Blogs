# iOS压缩和解压
好久没有写了，今天来聊聊iOS中的压缩和解压缩。
压缩其实在iOS中的运用算很少见的，所以很多小伙伴不是很了解，我也是工作多年第一次遇到，今天就来分享一下。
## 一、为什么要压缩
这个相信所有来看这篇文章的同学都有自己的不同答案，总的来说就一个目的：**减小数据包的大小，增加传输效率。**
## 二、压缩分类
iOS中的压缩主要可以分为两种：
###### 1.文件压缩。（例：.mp3 .jpg .plist）
这种运用最多，比如下载小视频，音乐和大图片等。节约流量，加快传输。
###### 2.数据流压缩。
这种需求，主要来此于有些页面，比如首页，一个接口包含了太多内容，通过压缩提高传输效率。
## 三、文件压缩
文件压缩建议使用**[ZipArchive](https://github.com/ZipArchive/ZipArchive)** git超过4000star，你值得拥有。
代码调用作者已经写的很清楚了，简单举个项目中使用的例子
```
        
        NSString *path,*zipPath;
    //获取资源数据
        NSData * data = [[response.data valueForKey:@"data"] dataUsingEncoding:NSUTF8StringEncoding];
            NSArray*pathes=NSSearchPathForDirectoriesInDomains(NSCachesDirectory,NSUserDomainMask,YES);
    //获取存储路径
        path= [pathes objectAtIndex:0];
        NSLog(@"%@",path);
        zipPath= [path stringByAppendingPathComponent:@"zipfile.zip"];
    //数据存储到本地
       [data writeToFile:zipPath options:0 error:nil];
    //解压
        [SSZipArchive unzipFileAtPath:zipPath toDestination:path progressHandler:^(NSString * _Nonnull entry, unz_file_info zipInfo, long entryNumber, long total) {
            
        } completionHandler:^(NSString * _Nonnull path, BOOL succeeded, NSError * _Nullable error) {
            
        }];
```

## 四、数据流压缩
这种压缩，通常是采用Gzip的格式压缩。
后台将前端需要的数据对象（通常是字典或者数组）转化成byte数组，然后压缩。
将压缩后的byte数组通过编码格式（**常用ISO 8859-1**）转化成字符串，传输。
对应的客户端，根据压缩，编码方式，相应进行解码。
这里解压使用的是 **[GZIP](https://github.com/nicklockwood/GZIP)**。
话不多说，上代码。
```
         //从接口获取字符串数据
        NSString *str = [response.data valueForKey:@"data"];
        //ISO 8859-1编码格式
        NSStringEncoding enc = CFStringConvertEncodingToNSStringEncoding(kCFStringEncodingISOLatin1);
        //字符串转化为byte数组
        NSData * dataGzip = [str dataUsingEncoding:enc];
        //解压
        NSData *unzip =  [dataGzip gunzippedData];
        
        NSError *err;
        //将解压得到的byte数组转换成json对象
        NSDictionary *dic = [NSJSONSerialization JSONObjectWithData:unzip options:NSJSONReadingMutableContainers error:&err];
```
