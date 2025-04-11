- DOCKER 是什么？
- 如何使用DOCKER 技术创建与环境无关的容器系统？
- DOCKERFILE 配置文件中的COPY 和ADD 指令有什么不同
- DOCKER 映像（IMAGE）是什么
- DOCKER 容器（CONTAINER）是什么
- DOCKER 中心（HUB）什么概念
- 在任意给定时间点指出一个DOCKER 容器可能存在的运行阶段
- 有什么方法确定一个DOCKER 容器运行状态
- 在DOCKERFILE 配置文件中最常用的指令有哪些
- 什么类型的应用（无状态性或有状态性）更适合DOCKER 容器技术
- 解释基本DOCKER 应用流程
- DOCKER IMAGE 和DOCKER LAYER (层) 有什么不同
- 虚拟化技术是什么
- 什么是孤儿卷及如何删除它
- docker为什么是轻量级别的
- docker run -v
- docker rm
- 虚拟机和普通容器的不同点？网络实现都有什么原理？
- docker中copy, add的区别
- docker中entrypoint和cmd的区别



# entrypoint和cmd的区别

在 Dockerfile 中，`CMD` 和 `ENTRYPOINT` 都是用于定义容器启动时执行的命令，但它们的行为和用途有显著区别。以下是具体对比：


### **一、核心功能与语法**
| 指令       | 功能                                                                 | 语法格式                                                                 | 默认行为                                                                 |
|------------|----------------------------------------------------------------------|--------------------------------------------------------------------------|--------------------------------------------------------------------------|
| **CMD**    | 设置容器启动时的**默认命令**，可被 `docker run` 的参数覆盖。         | <br>1. `CMD ["executable", "param1", "param2"]`（推荐，exec 格式）<br>2. `CMD command param1 param2`（shell 格式）<br>3. `CMD ["param1", "param2"]`（为 `ENTRYPOINT` 提供默认参数） | 一个 Dockerfile 中只能有一个 `CMD`，后出现的会覆盖前一个。               |
| **ENTRYPOINT** | 设置容器启动时的**入口程序**，通常不可被直接覆盖，参数会追加到其后。 | <br>1. `ENTRYPOINT ["executable", "param1", "param2"]`（推荐，exec 格式）<br>2. `ENTRYPOINT command param1 param2`（shell 格式）                                                          | 可多次定义，**仅最后一个有效**，且会忽略之前的 `CMD`（除非 `CMD` 为其提供参数）。 |


### **二、参数处理方式**
#### 1. **CMD：默认命令，可被覆盖**
- 当运行 `docker run <镜像> [参数]` 时，若提供了额外参数，会**直接覆盖 `CMD` 定义的命令**。  
  **示例**：  
  ```dockerfile
  CMD ["echo", "Hello, World!"]  # 默认命令
  ```  
  运行 `docker run myimage "Hi there"` 时，实际执行的是 `Hi there`，而非默认的 `echo` 命令。

#### 2. **ENTRYPOINT：固定入口，参数追加**
- 当运行 `docker run <镜像> [参数]` 时，提供的参数会**追加到 `ENTRYPOINT` 命令之后**，作为参数传递，不会覆盖入口程序。  
  **示例**：  
  ```dockerfile
  ENTRYPOINT ["echo"]  # 固定入口程序
  ```  
  运行 `docker run myimage "Hello, World!"` 时，实际执行 `echo "Hello, World!"`，参数被正确传递。


### **三、典型使用场景**
#### 1. **CMD 的常见场景**
- **设置默认命令**：当容器没有明确的固定入口程序时，如交互式容器（如 `bash` 终端）。  
  ```dockerfile
  CMD ["bash"]  # 默认启动 bash 终端，用户可通过 `docker run -it myimage sh` 覆盖
  ```  
- **为 `ENTRYPOINT` 提供默认参数**：当 `ENTRYPOINT` 定义固定程序，`CMD` 定义其默认参数（此时 `CMD` 必须为数组格式）。  
  ```dockerfile
  ENTRYPOINT ["curl"]  # 固定程序
  CMD ["https://www.example.com"]  # 默认参数，可被 `docker run myimage https://baidu.com` 覆盖
  ```

#### 2. **ENTRYPOINT 的常见场景**
- **固定不可变的主程序**：如运行一个脚本或可执行文件，不希望用户随意修改命令本身，仅允许修改参数。  
  ```dockerfile
  ENTRYPOINT ["/app/start.sh"]  # 必须执行该脚本，用户参数会追加到其后
  ```  
- **预处理环境或校验参数**：例如在启动应用前检查配置文件是否存在，或设置环境变量。  
  ```dockerfile
  ENTRYPOINT ["/entrypoint.sh"]  # 脚本内包含前置逻辑，再执行 `CMD` 的默认命令
  ```


### **四、组合使用（最佳实践）**
通常将 `ENTRYPOINT` 和 `CMD` 结合使用，实现“固定程序 + 灵活参数”的模式：  
1. **`ENTRYPOINT` 定义固定命令**（如程序路径）。  
2. **`CMD` 定义该命令的默认参数**（可被用户运行时修改）。  

**示例**：  
```dockerfile
# 定义固定入口程序为 `nginx`
ENTRYPOINT ["nginx", "-g", "daemon off;"]  
# 设置默认参数（监听端口），用户可通过 `docker run myimage -p 8080:80` 覆盖
CMD ["-h", "0.0.0.0"]  
```  
当运行 `docker run myimage -h 127.0.0.0` 时，实际执行：  
```bash
nginx -g "daemon off;" -h 127.0.0.0
```


### **五、关键区别总结**
| 对比项         | **CMD**                                  | **ENTRYPOINT**                          |
|----------------|------------------------------------------|-----------------------------------------|
| **目的**       | 设置默认命令，可被覆盖                   | 设置固定入口程序，参数追加不覆盖        |
| **参数处理**   | `docker run` 的参数直接覆盖默认命令       | `docker run` 的参数追加到命令之后        |
| **覆盖方式**   | 可通过 `docker run <镜像> [命令]` 覆盖     | 需通过 `--entrypoint` 选项显式修改入口   |
| **典型场景**   | 交互式容器、默认参数、临时命令            | 固定主程序、预处理脚本、不可变入口       |
| **组合使用**   | 常作为 `ENTRYPOINT` 的默认参数            | 常搭配 `CMD` 提供灵活参数                |


### **六、实战示例**
#### 1. 仅用 `CMD`（可被覆盖）：
```dockerfile
FROM alpine
CMD ["echo", "This is a default command."]
```  
运行：  
```bash
docker run myimage  # 输出：This is a default command.  
docker run myimage "Override!"  # 输出：Override!（覆盖原命令）
```

#### 2. 仅用 `ENTRYPOINT`（参数追加）：
```dockerfile
FROM alpine
ENTRYPOINT ["echo"]
CMD ["Default message."]  # 会被忽略（除非与 ENTRYPOINT 组合）
```  
运行：  
```bash
docker run myimage "Custom message."  # 输出：Custom message.（CMD 被忽略）
```

#### 3. 组合使用（推荐）：
```dockerfile
FROM alpine
ENTRYPOINT ["echo"]  # 固定程序
CMD ["Default message."]  # 默认参数
```  
运行：  
```bash
docker run myimage  # 输出：Default message.（使用 CMD 默认参数）  
docker run myimage "Custom message."  # 输出：Custom message.（参数覆盖 CMD）
```


### **总结**
- **选 `CMD`**：当需要设置一个可被用户轻松覆盖的默认命令（如交互式终端、临时任务）。  
- **选 `ENTRYPOINT`**：当需要固定容器启动时必须执行的程序（如服务、脚本），且允许用户传递参数而不改变主程序。  
- **最佳实践**：结合两者，用 `ENTRYPOINT` 定义固定入口，`CMD` 提供默认参数，兼顾灵活性与稳定性。

# copy和add 的区别
在 Docker 构建过程中，**缓存变化**指的是 Docker 镜像层的缓存状态因指令或上下文变化而失效的现象。这一机制直接影响镜像构建效率，理解其原理能帮助开发者优化构建流程。以下是详细解析：

### 一、缓存变化的本质
Docker 构建镜像时，会按顺序为每条指令生成一个**镜像层**，并将这些层缓存到本地。若后续构建时指令内容或上下文未变，Docker 会复用缓存层以加速构建。反之，若指令或依赖的文件发生变化，缓存将失效，需重新执行该指令及后续所有层。

例如：
```dockerfile
FROM alpine:latest
RUN apk add --no-cache bash  # 生成层A
COPY app /app                # 生成层B（依赖层A）
```
- **第一次构建**：执行两条指令，生成层A和层B。
- **第二次构建**：若`COPY`的`app`文件未变，直接复用层B；若`app`文件修改，则层B缓存失效，需重新执行。

### 二、触发缓存变化的核心因素
#### 1. **指令内容变化**
- **指令本身修改**：例如将`RUN apk add bash`改为`RUN apk add bash curl`，会导致缓存失效。
- **参数或路径变化**：`COPY ./src /app`改为`COPY ./src2 /app`，目标路径未变但源路径变化，缓存失效。

#### 2. **文件内容变化**
- **COPY/ADD 的源文件**：Docker 通过**文件内容的哈希值**判断是否变化。即使文件名未变，内容修改也会触发缓存失效。
- **构建上下文变化**：若`.dockerignore`未排除某些文件，新增文件会导致上下文哈希值变化，所有依赖该上下文的指令（如`COPY`）缓存失效。

#### 3. **指令顺序调整**
Docker 按指令顺序构建镜像层，调整顺序会导致后续层的依赖关系变化。例如：
```dockerfile
# 原顺序
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# 调整后
COPY . .
RUN pip install -r requirements.txt
```
调整后，`RUN`指令的上下文包含全部代码，缓存失效概率大幅增加。

#### 4. **基础镜像变化**
若`FROM`指令的基础镜像版本更新（如`alpine:3.16`→`alpine:3.17`），所有后续层的缓存均失效。

#### 5. **环境变量变化**
`ENV`指令定义的变量会影响后续指令。例如：
```dockerfile
ENV VERSION=1.0
RUN apt-get install myapp=$VERSION  # 依赖VERSION变量
```
若`VERSION`改为`2.0`，`RUN`指令的缓存失效。

### 三、COPY 与 ADD 在缓存变化中的差异
| 指令   | 缓存行为                                                                 | 示例与影响                                                                 |
|--------|--------------------------------------------------------------------------|----------------------------------------------------------------------------|
| **COPY** | 仅当源文件内容或目标路径变化时失效，缓存稳定性高。                          | `COPY app.tar.gz /app`：若文件内容未变，直接复制；若修改，缓存失效。         |
| **ADD**  | 以下情况均可能触发缓存失效：<br>- 源文件内容/路径变化<br>- 远程URL内容变化<br>- 压缩文件解压后的内容变化（隐式解压时） | `ADD https://example.com/app.tar.gz /app`：每次构建均下载，缓存失效。 |

**推荐实践**：优先使用`COPY`，仅在必要时（如本地压缩文件自动解压）使用`ADD`，以减少缓存失效风险。

### 四、缓存变化的影响与优化策略
#### 1. **构建效率**
- **缓存命中**：复用层可减少重复下载和计算，例如`RUN apt-get update`的缓存复用可节省数秒至数分钟。
- **缓存失效**：若频繁修改代码，可能导致多层缓存失效，构建时间显著增加。

#### 2. **镜像体积**
缓存失效可能导致中间层冗余。例如：
```dockerfile
RUN apt-get update && apt-get install -y nginx  # 层A
RUN apt-get install -y curl                    # 层B（依赖层A）
```
若修改层B为`RUN apt-get install -y wget`，层B缓存失效，但层A仍保留，最终镜像包含层A和新层B。

#### 3. **优化策略**
- **分层构建**：将稳定的依赖（如安装包）放在前面，易变的代码放在后面。
  ```dockerfile
  FROM python:3.9
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt  # 缓存复用
  COPY . .
  ```
- **使用.dockerignore**：排除不必要的文件，减少上下文变化。
- **手动控制缓存**：
  - `--no-cache`：强制禁用缓存（如`docker build --no-cache -t myapp .`）。
  - `--cache-from`：指定缓存来源（如`docker build --cache-from myapp:latest -t myapp:new .`）。

### 五、典型场景分析
#### 场景1：代码频繁修改
- **问题**：每次修改代码都导致`COPY . .`缓存失效，需重新安装依赖。
- **优化**：将依赖与代码分离。
  ```dockerfile
  FROM node:18
  WORKDIR /app
  COPY package*.json .
  RUN npm ci  # 缓存复用
  COPY . .
  ```

#### 场景2：远程依赖更新
- **问题**：`ADD`下载的文件内容变化，但未触发缓存失效。
- **优化**：改用`curl`下载并校验哈希值。
  ```dockerfile
  RUN curl -fsSL "https://example.com/app.tar.gz" -o app.tar.gz \
    && echo "hash  app.tar.gz" | sha256sum -c -  # 哈希校验
  ```

### 六、总结
缓存变化是 Docker 构建机制的核心，其本质是**镜像层的动态更新**。开发者需通过合理组织 Dockerfile、优化指令顺序、控制上下文变化等手段，平衡构建效率与镜像一致性。理解缓存变化的原理，能帮助快速定位构建问题，并针对性地进行性能优化。