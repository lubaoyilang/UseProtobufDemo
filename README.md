# UseProtobufDemo

# Require
这是约定的数据格式：
![](https://github.com/HelloiWorld/UseProtobufDemo/blob/master/002If1Mfzy77mN5b9lR47%26690.jpeg)

## Http + AFNetworking 3.0 + Protobuf
    Test_Req *req = [Test_Req new];
    req.param1 = @"0";
    req.param2 = 1;
    [self requestProtobufCommandId:CommandEnum_CmdTest params:[req data] playerId:0 sessionId:0 completionHandler:^(NSData *data) {
        if (data) {
            Test_Rsp *rsp = [Test_Rsp parseFromData:data error:nil];
            NSLog(@"rsp: %@, result1:%@, result2: %lld", rsp,rsp.result1,rsp.result2);
        }
    } errorHandler:^(int32_t errorCode) {
        NSLog(@"errorCode: %d",errorCode);
    }];
    
    // Http+Protobuf
    - (void)requestProtobufCommandId:(int)commandId
                          params:(NSData *)params
                        playerId:(uint64_t)playerID
                       sessionId:(uint64_t)sessionID
               completionHandler:(void(^)(NSData *data))dataBlock
                    errorHandler:(void(^)(int32_t errorCode))errorBlock {
    //  苹果 整数字节使用小端序传输，而其他都是网络端序大端序传输
    //  HTONS 转换端序，从小端序转为大端序，HTONL 转化4字节端序，HTONLL转化8字节端序。
    //  int  htonl ->short 类型的主机存储－>网络的网络存储，并写入内存块
    //  char,string 类型不需要转换
    
    NSMutableData *protobufData = [[NSMutableData alloc] init];
    // 0XFF
    int str = 0xff;
    str = htonl(str);
    [protobufData appendBytes:&str length:sizeof(str)];
    // playerId
    uint64_t playerId = 0;
    playerId = htonll(playerId);
    NSData *playerIdData = [NSData dataWithBytes: &playerId length: sizeof(playerId)];
    [protobufData appendData:playerIdData];
    // sessionId
    uint64_t sessionId = 0;
    sessionId = htonll(sessionId);
    NSData *sessionIdData = [NSData dataWithBytes: &sessionId length: sizeof(sessionId)];
    [protobufData appendData:sessionIdData];
    // size
    u_long size = params.length+4;
    size = htonl(size);
    [protobufData appendBytes:&size length:4];
    // commandId
    commandId = htonl(commandId);
    NSData *commandIdData = [NSData dataWithBytes: &commandId length: sizeof(commandId)];
    [protobufData appendData:commandIdData];
    // data
    [protobufData appendData:params];
    
    Byte *byte = (Byte *)[protobufData bytes];
    NSString *byteString = @"";
    for (int i=0 ; i<[protobufData length]; i++) {
        byteString = [byteString stringByAppendingString:[NSString stringWithFormat:@"%d ",byte[i]]];
    }
    NSLog(@"byteString: %@",byteString);
    
    //第一步，创建url
    NSURL *url = [NSURL URLWithString:@"192.168.1.1:8080"];
    //第二步，创建请求
    NSMutableURLRequest *request = [[NSMutableURLRequest alloc]initWithURL:url cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:10];
    [request setHTTPMethod:@"POST"];
    [request setHTTPBody:protobufData];
    //第三步，连接服务器
    AFHTTPSessionManager *_manager = [AFHTTPSessionManager manager];
    _manager.responseSerializer = [AFHTTPResponseSerializer serializer];
    
    NSURLSessionDataTask *task = [_manager dataTaskWithRequest:request completionHandler:^(NSURLResponse * _Nonnull response, id  _Nullable responseObject, NSError * _Nullable error) {
        if (error){
            NSLog(@"error = %@",error);
            dataBlock(nil);
        }else{
            NSLog(@"protobuf data = %@", responseObject);
            
            NSData *data = responseObject;
            if (data.length > 12) {
                // normal data
                dataBlock([data subdataWithRange:NSMakeRange(12, data.length-12)]);
            } else {
                dataBlock(nil);
            }
        }
    }];
    [task resume];
    }

## CocoaAsyncSocket + TCP + Protobuf
    //初始化socket并连接到服务端
    - (void)initSocket {
        GCDAsyncSocket *gcdSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
    }

    #pragma mark - 对外的一些接口
    //建立连接
    - (BOOL)connect {
        return [gcdSocket connectToHost:@"192.168.1.1" onPort:8080 error:nil];
    }

    //断开连接
    - (void)disConnect {
        [gcdSocket disconnect];
    }

    #pragma mark - GCDAsyncSocketDelegate
    //连接成功调用
    - (void)socket:(GCDAsyncSocket *)sock didConnectToHost:(NSString *)host port:(uint16_t)port {
        NSLog(@"连接成功,host:%@,port:%d",host,port);
        //监听读数据的代理  -1永远监听，不超时，但是只收一次消息，
        //所以每次接受到消息还得调用一次
        [gcdSocket readDataWithTimeout:-1 tag:0];
        //心跳写在这...
    }

    //断开连接的时候调用
    - (void)socketDidDisconnect:(GCDAsyncSocket *)sock withError:(nullable NSError *)err {
        NSLog(@"断开连接,host:%@,port:%d",sock.localHost,sock.localPort);
        //断线重连写在这...
        if (err) {
            //再次重连
      } else {
       //正常断开
      }
    }
    
## Additional
例如游戏，有时一些额外的事件也会附加在正式协议数据后面传递过来，这时你需要根据数据长度将其分段，然后处理资源变化及全局事件

    + (void)handleResponseAdditional:(NSData *)additionalData {
    __block NSMutableData *mutableData = [NSMutableData dataWithData:additionalData];
    while (additionalData.length > 12) {
        NSData *sizeData = [mutableData subdataWithRange:NSMakeRange(4, 4)];
        int i;
        [sizeData getBytes: &i length: sizeof(i)];
        i = htonl(i);
        
        NSData *cmdIdData = [mutableData subdataWithRange:NSMakeRange(8, 4)];
        int j;
        [cmdIdData getBytes: &j length: sizeof(j)];
        j = htonl(j);
        
        // additional event command Id
        if (j == CommandEnum_CmdDefault) {
            
        } else {
            
        }
        
        if ((int)mutableData.length-8-i > 0) {
            [mutableData setData:[mutableData subdataWithRange:NSMakeRange(8+i, mutableData.length-8-i)]];
        } else {
            break;
        }
    }
    } 
    
    
## PBParser解析器
类似JSON格式化工具，用于快速为本地model装载数据以及本地字段与后台不匹配的情况。

    #import "NSObject+DataMerge.h"
    #import "NSObject+ProtobufExtension.h"
    
#### Set up model 
    
    Model *model = [[Model alloc] init]; 
    [model setupWithObject:[Model instanceWithProtoObject:rsp]];
    
#### Map / Replace 

    #pragma mark- Map
    + (NSDictionary *)replacedPropertyKeypathsForProtobuf {
       return @{@"resultStr" : @"result1",
                @"resultNum" : @"result2"};
    }
    
    
## Shell Command
批量生成phobjc文件的一个小脚本，将Module&Src文件下的所有proto文件编译并输出 

    #!/bin/bash     
    
    BASEDIR=$(dirname "$0") 
    cd "$BASEDIR"

    # relative url
    rm -rf ./Module\&Src/*.pbobjc.*

    protoc -I=./Module\&Src --objc_out=./Module\&Src ./Module\&Src/*.proto
  
