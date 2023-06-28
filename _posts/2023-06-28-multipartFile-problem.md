---
layout: post

title: 组装MultipartFile问题记录

categories: [Java]


---



### 问题记录：自行组装MultipartFile发生的问题

#### 前提：

MultipartFile是网页上传文件流需要使用到的，包含在StandardMultipartHttpServletRequest里由parseRequest处理，这个流程在mvc架构里面在前端获取到MultipartFile已经由架构给你处理好可用的状态了。

#### 操作: 

自己组装MultipartFile上传文件，搜索到可以使用springtest提供的MockMultipartFile来生成MultipartFile代码如下所示：

```
InputStream inputStream = getClass().getResourceAsStream("/path/to/file");

// 创建一个临时文件
File tempFile = File.createTempFile("temp", null);
tempFile.deleteOnExit();

// 将 InputStream 写入临时文件
try (OutputStream outputStream = new FileOutputStream(tempFile)) {
    byte[] buffer = new byte[1024];
    int bytesRead;
    while ((bytesRead = inputStream.read(buffer)) != -1) {
        outputStream.write(buffer, 0, bytesRead);
    }
}

// 创建一个 MultipartFile 对象
MultipartFile multipartFile = new MockMultipartFile(tempFile.getName(), tempFile.getName(),
        Files.probeContentType(tempFile.toPath()), new FileInputStream(tempFile));

// 创建包含单个 MultipartFile 的 List
List<MultipartFile> multipartFiles = new ArrayList<>();
multipartFiles.add(multipartFile);

```

使用MockMultipartFile生成实例然后上传文件，报错(大概信息是负载均衡相关与上传文件无关联)：

```
2023-06-27 16:13:52.840  INFO 8 --- [hoerodon-file-1] f.a.AutowiredAnnotationBeanPostProcessor : JSR-330 'javax.inject.Inject' annotation found and supported for autowiring
2023-06-27 16:13:53.005  INFO 8 --- [hoerodon-file-1] c.netflix.config.ChainedDynamicProperty  : Flipping property: choerodon-file.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
2023-06-27 16:13:53.019  INFO 8 --- [hoerodon-file-1] c.n.u.concurrent.ShutdownEnabledTimer    : Shutdown hook installed for: NFLoadBalancer-PingTimer-choerodon-file
2023-06-27 16:13:53.021  INFO 8 --- [hoerodon-file-1] c.netflix.loadbalancer.BaseLoadBalancer  : Client: choerodon-file instantiated a LoadBalancer: DynamicServerListLoadBalancer:{NFLoadBalancer:name=choerodon-file,current list of Servers=[],Load balancer stats=Zone stats: {},Server stats: []}ServerList:null
2023-06-27 16:13:53.023  INFO 8 --- [hoerodon-file-1] c.n.l.DynamicServerListLoadBalancer      : Using serverListUpdater PollingServerListUpdater
2023-06-27 16:13:53.024  INFO 8 --- [hoerodon-file-1] c.netflix.config.ChainedDynamicProperty  : Flipping property: choerodon-file.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
2023-06-27 16:13:53.025  INFO 8 --- [hoerodon-file-1] c.n.l.DynamicServerListLoadBalancer      : DynamicServerListLoadBalancer for client choerodon-file initialized: DynamicServerListLoadBalancer:{NFLoadBalancer:name=choerodon-file,current list of Servers=[10.0.0.19:8110],Load balancer stats=Zone stats: {defaultzone=[Zone:defaultzone;    Instance count:1;       Active connections count: 0;    Circuit breaker tripped count: 0;       Active connections per server: 0.0;]
```

查看到请求的HttpServletRequest instace of StandardMultipartHttpServletRequest，在类里面找到静态类StandardMultipartFile，依照它的方法大概实现一个CustomMultipartFile

```
package io.choerodon.kb.infra.entity;

/**
 * MultipartFile接口实现
 *
 * @author liyh
 * @date 2023/06/27
 */
import org.springframework.lang.Nullable;
import org.springframework.util.Assert;
import org.springframework.util.FileCopyUtils;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.io.File;
import java.nio.file.Files;

public class CustomMultipartFile implements MultipartFile {
    private final String name;
    private String originalFilename;
    @Nullable
    private String contentType;
    private final InputStream inputStream;

    public CustomMultipartFile(String name, @Nullable String originalFilename, @Nullable String contentType, @Nullable InputStream inputStream) {
        Assert.hasLength(name, "Name must not be null");
        this.name = name;
        this.originalFilename = originalFilename != null ? originalFilename : "";
        this.contentType = contentType;
        this.inputStream = inputStream;
    }

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public String getOriginalFilename() {
        return this.originalFilename;
    }

    @Nullable
    @Override
    public String getContentType() {
        return this.contentType;
    }

    @Override
    public boolean isEmpty() {
        try {
            return this.inputStream.available()==0;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return false;
    }

    @Override
    public long getSize() {
        try {
            return Long.valueOf(this.inputStream.available());
        } catch (IOException e) {
            e.printStackTrace();
        }
        return 0;
    }

    @Override
    public byte[] getBytes() throws IOException {
        return FileCopyUtils.copyToByteArray(this.inputStream);
    }

    @Override
    public InputStream getInputStream() throws IOException {
        return new BufferedInputStream(inputStream);
    }

    @Override
    public void transferTo(File file) throws IOException, IllegalStateException {
        if (file.isAbsolute() && !file.exists()) {
            FileCopyUtils.copy(this.getInputStream(), Files.newOutputStream(file.toPath()));
        }
    }


}

```

运行之后发现报同样的错误，仔细去查看part里面封装了很多由前端过来的参数，没办法去一一设置，放弃构造StandardMultipartFile。

#### 解决:

通过把模板上传到minio调用接口进行该文件的复制实现文件模板的上传

```
fileClient.copyFileByUrl(organization,fileurl,buket,buket,fileName);
```

ps：实现的时候考虑问题比较欠缺，按照上传文件的流程来写创建问题被束缚到了浪费了很多时间，如果先去查看底层提供的方法，通过方法组合找到一个最佳的实现方案，这种思路效率会高很多。