# Fabric-gm 国密实验复现与部署指南（针对 Ubuntu 服务器）

> 本指南基于《Hyperledger Fabric 平台的国密算法嵌入研究》（曹琪等，2021）论文，指导在 Ubuntu 服务器上完整复现论文中图 10~图 17 的全部实验结果。

---

## 一、环境准备

### 1.1 系统基础环境

**操作系统要求**：Ubuntu 16.04/18.04/20.04 LTS x86_64（论文实验环境为 Ubuntu 16.04.05）

```bash
# 查看系统版本
cat /etc/os-release
uname -a

# 更新系统包
sudo apt-get update && sudo apt-get upgrade -y

# 安装基础依赖
sudo apt-get install -y \
    build-essential \
    git \
    curl \
    wget \
    unzip \
    gcc \
    g++ \
    make \
    libtool \
    libltdl-dev \
    python3 \
    python3-pip \
    jq \
    tree \
    net-tools \
    software-properties-common \
    apt-transport-https \
    ca-certificates
```

### 1.2 安装 Go 语言环境（Go 1.14+）

fabric-gm 项目的 `go.mod` 指定 `go 1.14`，建议安装 Go 1.14.x 或 Go 1.15.x 版本。

```bash
# 下载 Go 1.15.15（推荐版本）
cd /tmp
wget https://golang.org/dl/go1.15.15.linux-amd64.tar.gz

# 解压安装
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.15.15.linux-amd64.tar.gz

# 配置环境变量（追加到 ~/.bashrc 末尾）
cat >> ~/.bashrc << 'GOEOF'

# Go 语言环境变量
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$GOROOT/bin:$GOPATH/bin:$PATH
export GO111MODULE=on
export GOPROXY=https://goproxy.cn,direct
GOEOF

# 使环境变量立即生效
source ~/.bashrc

# 验证安装
go version
# 预期输出: go version go1.15.15 linux/amd64

go env GOROOT GOPATH GO111MODULE GOPROXY
```

### 1.3 安装 Docker 与 docker-compose

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sudo sh

# 将当前用户加入 docker 组（避免每次使用 sudo）
sudo usermod -aG docker $USER

# 注意：需要重新登录终端使 docker 组生效，或执行：
newgrp docker

# 验证 Docker 安装
docker version
docker info

# 安装 docker-compose（1.25.x 版本，兼容 Fabric 2.2）
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 验证 docker-compose
docker-compose version
# 预期输出: docker-compose version 1.25.5, build ...
```

### 1.4 安装 GmSSL（用于验证国密算法结果）

```bash
# 克隆 GmSSL 源码
cd /tmp
git clone https://github.com/guanzhi/GmSSL.git
cd GmSSL

# 编译安装
./config
make -j$(nproc)
sudo make install

# 验证安装
gmssl version
# 预期输出: GmSSL x.x.x ...

# 如果提示找不到共享库，执行：
sudo ldconfig
```

### 1.5 克隆 fabric-gm 项目源码

```bash
# 创建工作目录
mkdir -p $GOPATH/src/github.com/ponyletter
cd $GOPATH/src/github.com/ponyletter

# 克隆 fabric-gm 仓库
git clone https://github.com/ponyletter/fabric-gm.git
cd fabric-gm

# 查看项目结构
ls -la
tree -L 1

# 确认 go.mod 中的 module 路径
head -5 go.mod
# 预期输出第一行: module github.com/ponyletter/fabric-gm
```

---

## 二、编译 Fabric-gm 二进制文件

### 2.1 编译前准备

```bash
# 进入项目根目录
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 确保 vendor 目录完整（项目已包含 vendor）
ls vendor/

# 检查 tjfoc/gmsm 国密库
ls tjfoc/gmsm/
# 预期看到: sm2/ sm3/ sm4/ vendor/ go.mod 等

# 查看 Makefile 支持的编译目标
grep -E "^[a-zA-Z_-]+:" Makefile | head -20
```

### 2.2 编译 peer 节点

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 编译 peer 二进制
make peer

# 验证编译结果
ls -la build/bin/peer
./build/bin/peer version
```

### 2.3 编译 orderer 排序节点

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 编译 orderer 二进制
make orderer

# 验证编译结果
ls -la build/bin/orderer
./build/bin/orderer version
```

### 2.4 编译 cryptogen 证书生成工具

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 编译 cryptogen 二进制
make cryptogen

# 验证编译结果
ls -la build/bin/cryptogen
./build/bin/cryptogen version

# 同时编译其他工具
make configtxgen
make configtxlator

# 验证
ls -la build/bin/
```

### 2.5 编译全部工具（一次性）

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 编译全部本地二进制（peer, orderer, cryptogen, configtxgen, configtxlator, idemixgen, discover）
make native

# 验证所有二进制文件
ls -la build/bin/
```

### 2.6 构建 Docker 镜像

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 构建全部 Fabric-gm Docker 镜像
make docker

# 查看生成的镜像列表
docker images | grep -E "hyperledger|fabric"

# 预期应看到以下镜像：
# hyperledger/fabric-peer
# hyperledger/fabric-orderer
# hyperledger/fabric-tools
# hyperledger/fabric-ccenv
# hyperledger/fabric-baseos

# 为镜像打上 latest 标签（如果需要）
make docker-tag-latest
```

### 2.7 拉取第三方依赖镜像

```bash
# 拉取 Kafka、Zookeeper、CouchDB 等第三方镜像
make docker-thirdparty

# 或手动拉取
docker pull hyperledger/fabric-couchdb:latest
docker pull hyperledger/fabric-kafka:latest
docker pull hyperledger/fabric-zookeeper:latest
```

---

## 三、复现图 10~图 13：BCCSP 国密算法接口单元测试

### 3.1 复现图 10：SM3 哈希接口调用可用性验证

**执行目录**：`$GOPATH/src/github.com/ponyletter/fabric-gm`

**对应论文图片**：图 10 Fabric-gm 上的 SM3 哈希接口调用可用性验证

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 运行 bccsp 包的单元测试，验证默认哈希选项指向 SM3
go test -v ./bccsp/... -run TestSM3 2>&1 | tee /tmp/sm3_usability_test.log

# 也可运行 bccsp/gm 包下的全部测试
go test -v ./bccsp/gm/... 2>&1 | tee /tmp/bccsp_gm_test.log

# 查看 factory 默认选项测试（验证 ProviderName 为 "GM"）
go test -v ./bccsp/factory/... -run TestGetDefaultOpts 2>&1 | tee /tmp/factory_test.log

# 关键检查：确认默认 BCCSP 实例为 GM，HashFamily 为 GMSM3
grep -n "GM\|GMSM3\|SM3" /tmp/factory_test.log
```

**预期输出**：测试通过（PASS），日志中显示默认的哈希选项为 SM3 哈希算法接口。与论文图 10 中长方形框内显示的内容对应。

### 3.2 复现图 11：SM3 哈希接口有效性验证

**执行目录**：`$GOPATH/src/github.com/ponyletter/fabric-gm`

**对应论文图片**：图 11 Fabric-gm 上的 SM3 哈希接口有效性验证

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 步骤 1：使用 GmSSL 计算 "Hello World" 的 SM3 哈希值（对应图 11(a)）
echo -n "Hello World" | gmssl sm3
# 预期输出：SM3 哈希值（64 位十六进制字符串）

# 步骤 2：编写并运行 Go 测试脚本验证 BCCSP SM3 接口（对应图 11(b)）
# 创建测试文件
cat > /tmp/sm3_validity_test.go << 'EOF'
package main

import (
    "encoding/hex"
    "fmt"
    "github.com/tjfoc/gmsm/sm3"
)

func main() {
    msg := []byte("Hello World")
    h := sm3.New()
    h.Write(msg)
    hash := h.Sum(nil)
    fmt.Printf("SM3 Hash of 'Hello World': %s\n", hex.EncodeToString(hash))
    fmt.Printf("Hash length: %d bytes\n", len(hash))
}
EOF

# 也可以直接运行 bccsp/gm 包中的哈希测试
go test -v ./bccsp/gm/... -run TestHash 2>&1 | tee /tmp/sm3_validity_result.log

# 对比两个哈希值是否一致
echo "=== 对比结果 ==="
echo "GmSSL 计算结果和 BCCSP SM3 接口计算结果应完全一致"
```

**预期输出**：GmSSL 和 Fabric-gm BCCSP 对同一消息 "Hello World" 计算的 SM3 哈希值完全一致。与论文图 11 中两组哈希结果对应。

### 3.3 复现图 12：SM4 算法接口可用性和有效性验证

**执行目录**：`$GOPATH/src/github.com/ponyletter/fabric-gm`

**对应论文图片**：图 12 Fabric-gmBCCSP 上的 SM4 算法接口可用性和有效性验证结果

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 运行 SM4 相关的单元测试
go test -v ./bccsp/gm/... -run TestSM4 2>&1 | tee /tmp/sm4_test.log

# 运行完整的加解密测试（KeyGen -> Encrypt -> Decrypt）
go test -v ./bccsp/gm/... -run TestEncryptDecrypt 2>&1 | tee /tmp/sm4_encrypt_decrypt.log

# 查看测试结果
cat /tmp/sm4_test.log
cat /tmp/sm4_encrypt_decrypt.log
```

**预期输出**：测试通过（PASS），SM4 密钥生成成功，加密后解密所得数据与原始明文一致。与论文图 12 对应。

### 3.4 复现图 13：SM2 算法接口可用性和有效性验证

**执行目录**：`$GOPATH/src/github.com/ponyletter/fabric-gm`

**对应论文图片**：图 13 Fabric-gmBCCSP 上的 SM2 算法接口有用性和有效性验证结果

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 运行 SM2 相关的单元测试
go test -v ./bccsp/gm/... -run TestSM2 2>&1 | tee /tmp/sm2_test.log

# 运行签名验签测试（KeyGen -> Sign -> Verify）
go test -v ./bccsp/gm/... -run TestSignVerify 2>&1 | tee /tmp/sm2_sign_verify.log

# 查看详细的签名 R 值和验签 r 值对比
cat /tmp/sm2_test.log
cat /tmp/sm2_sign_verify.log
```

**预期输出**：测试通过（PASS），签名过程 R 值与验签过程 r 值一致，SM2 签名验签接口可用且有效。与论文图 13 对应。

### 3.5 运行 BCCSP 全部单元测试（综合验证）

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 运行 bccsp/gm 包下的全部单元测试
go test -v -count=1 ./bccsp/gm/... 2>&1 | tee /tmp/bccsp_gm_all_tests.log

# 运行 bccsp 包（含 factory）的全部测试
go test -v -count=1 ./bccsp/... 2>&1 | tee /tmp/bccsp_all_tests.log

# 统计测试结果
echo "=== 测试统计 ==="
grep -c "PASS" /tmp/bccsp_gm_all_tests.log
grep -c "FAIL" /tmp/bccsp_gm_all_tests.log
grep "^ok\|^FAIL" /tmp/bccsp_gm_all_tests.log
```

---

## 四、复现图 14：改造后 cryptogen 生成国密证书

### 4.1 编译国密版 cryptogen

**执行目录**：`$GOPATH/src/github.com/ponyletter/fabric-gm`

**对应论文图片**：图 14 fabric-gmcryptogen 支持国密证书生成

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 编译 cryptogen
make cryptogen

# 确认生成
ls -la build/bin/cryptogen
```

### 4.2 准备证书生成配置文件

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 查看已有的 crypto-config.yaml 示例
find . -name "crypto-config*" -type f 2>/dev/null

# 如果没有现成的配置文件，创建一个标准的测试网络配置
mkdir -p /tmp/fabric-gm-test
cat > /tmp/fabric-gm-test/crypto-config.yaml << 'EOF'
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    EnableNodeOUs: true
    Specs:
      - Hostname: orderer

PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org2
    Domain: org2.example.com
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
EOF
```

### 4.3 使用国密版 cryptogen 生成证书

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 使用 cryptogen 生成国密证书
./build/bin/cryptogen generate \
    --config=/tmp/fabric-gm-test/crypto-config.yaml \
    --output=/tmp/fabric-gm-test/crypto-config \
    2>&1 | tee /tmp/cryptogen_output.log

# 查看生成的证书目录结构
tree /tmp/fabric-gm-test/crypto-config/ -L 4

# 查看某个节点证书的详细信息（使用 GmSSL 解析）
CERT_PATH=$(find /tmp/fabric-gm-test/crypto-config -name "signcerts" -path "*/peers/*" | head -1)
echo "证书路径: $CERT_PATH"

# 使用 GmSSL 查看证书内容，验证签名算法为 SM2、哈希算法为 SM3
gmssl x509 -in "$CERT_PATH"/*.pem -text -noout 2>&1 | tee /tmp/gm_cert_info.log

# 关键检查：签名算法应指向 SM2/SM3
grep -i "signature\|algorithm\|sm2\|sm3" /tmp/gm_cert_info.log
```

**预期输出**：证书中的签名算法指向 SM2，哈希算法指向 SM3。与论文图 14 对应。

---

## 五、复现图 15：Fabric-CA-gm 动态证书申请完整流程

### 5.1 构建 Fabric-CA-gm

**说明**：Fabric-CA-gm 是独立于 Fabric-gm 主仓库的组件。如果项目中包含 Fabric-CA-gm 源码，按以下步骤编译；否则需要单独克隆 Fabric-CA-gm 仓库。

```bash
# 如果项目中包含 fabric-ca-gm
# cd $GOPATH/src/github.com/ponyletter/fabric-ca-gm
# make fabric-ca-server
# make fabric-ca-client

# 如果使用 Docker 镜像方式，构建 CA 镜像
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 查看是否有 CA 相关的 Docker 构建
find . -path "*/ca/*Dockerfile*" -o -name "*ca*docker*" 2>/dev/null
```

### 5.2 启动 Fabric-CA-gm 服务器

```bash
# 创建 CA 工作目录
mkdir -p /tmp/fabric-ca-gm-test
cd /tmp/fabric-ca-gm-test

# 启动 Fabric-CA-gm 服务器（Docker 方式）
# 如果已构建了支持国密的 CA 镜像：
docker run -d \
    --name fabric-ca-gm-server \
    -p 7054:7054 \
    -v /tmp/fabric-ca-gm-test:/etc/hyperledger/fabric-ca-server \
    hyperledger/fabric-ca:latest \
    fabric-ca-server start -b admin:adminpw

# 等待 CA 服务器启动
sleep 5
docker logs fabric-ca-gm-server 2>&1 | tail -20

# 验证 CA 服务器正常运行
curl -s http://localhost:7054/cainfo | jq .
```

### 5.3 注册新用户并验证国密证书

**对应论文图片**：图 15 Fabric-CA-gm 支持国密证书生成

```bash
# 步骤 1：使用 fabric-ca-client 注册管理员
export FABRIC_CA_CLIENT_HOME=/tmp/fabric-ca-gm-test/admin

fabric-ca-client enroll \
    -u http://admin:adminpw@localhost:7054 \
    --tls.certfiles /tmp/fabric-ca-gm-test/tls-cert.pem \
    2>&1 | tee /tmp/ca_admin_enroll.log

# 步骤 2：注册名为 JimTest 的新用户（与论文一致）
fabric-ca-client register \
    --id.name JimTest \
    --id.secret JimTestpw \
    --id.type client \
    --id.affiliation org1.department1 \
    -u http://localhost:7054 \
    2>&1 | tee /tmp/ca_register_jimtest.log

# 步骤 3：以 JimTest 身份注册（获取证书）
export FABRIC_CA_CLIENT_HOME=/tmp/fabric-ca-gm-test/JimTest

fabric-ca-client enroll \
    -u http://JimTest:JimTestpw@localhost:7054 \
    2>&1 | tee /tmp/ca_jimtest_enroll.log

# 步骤 4：查看 JimTest 的证书信息
JIMTEST_CERT=$(find /tmp/fabric-ca-gm-test/JimTest -name "cert.pem" | head -1)
echo "JimTest 证书路径: $JIMTEST_CERT"

# 使用 GmSSL 展示证书信息
gmssl x509 -in "$JIMTEST_CERT" -text -noout 2>&1 | tee /tmp/jimtest_cert_info.log

# 关键检查：签名算法应指向 SM2、哈希算法应指向 SM3
grep -i "signature\|algorithm\|sm2\|sm3\|public.key" /tmp/jimtest_cert_info.log
```

**预期输出**：JimTest 用户的证书签名算法指向国密 SM2、哈希算法指向 SM3。与论文图 15 对应。

### 5.4 清理 CA 测试环境

```bash
# 停止并删除 CA 容器
docker stop fabric-ca-gm-server
docker rm fabric-ca-gm-server
```

---

## 六、复现图 16：byfn.sh 启动完整测试网络

### 6.1 准备测试网络

**执行目录**：`$GOPATH/src/github.com/ponyletter/fabric-gm`

**对应论文图片**：图 16 Fabric-gm 联盟链测试网络正常启动验证

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 查找 byfn.sh 脚本位置
find . -name "byfn.sh" -type f 2>/dev/null
# 通常位于 scripts/ 或 integration/e2e/ 目录下

# 查看脚本帮助
# ./scripts/byfn.sh --help
# 或
# 如果在其他路径
BYFN_SCRIPT=$(find . -name "byfn.sh" -type f | head -1)
echo "byfn.sh 路径: $BYFN_SCRIPT"
```

### 6.2 确保 Docker 镜像就绪

```bash
# 确认所有必要的 Docker 镜像已构建
docker images | grep hyperledger

# 必要镜像列表：
# hyperledger/fabric-peer
# hyperledger/fabric-orderer
# hyperledger/fabric-tools
# hyperledger/fabric-ccenv
# hyperledger/fabric-ca (如果使用 -a 参数启动 CA)
```

### 6.3 启动 Fabric-gm 测试网络

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 清理之前的网络（如果存在）
./byfn.sh down 2>/dev/null

# 启动测试网络（包含 CA 容器）
# -a 参数表示同时启动 CA 容器
./byfn.sh up -a 2>&1 | tee /tmp/byfn_startup.log

# 该命令依次执行：
# 1. 生成证书（调用 cryptogen）
# 2. 生成创世块和通道配置事务
# 3. 创建节点容器（包括 CA 容器）
# 4. 创建通道并将节点加入通道
# 5. 安装链码
# 6. 实例化链码
# 7. 执行一次完整的交易查询

# 检查输出中的 "END" 标志
tail -20 /tmp/byfn_startup.log
grep -i "END\|All GOOD\|SUCCESS" /tmp/byfn_startup.log
```

**预期输出**：脚本成功执行，输出显示 "END" 或 "All GOOD" 标志，表明 Fabric-gm 联盟链测试网络正常启动。与论文图 16 对应。

### 6.4 查看运行中的容器

```bash
# 查看所有 Fabric-gm 相关的运行容器
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 预期看到：
# peer0.org1.example.com
# peer1.org1.example.com
# peer0.org2.example.com
# peer1.org2.example.com
# orderer.example.com
# ca_org1 / ca_org2 (如果使用了 -a 参数)
```

### 6.5 关闭测试网络

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 关闭并清理测试网络
./byfn.sh down

# 清理所有链码容器和镜像
docker rm -f $(docker ps -aq --filter "name=dev-peer") 2>/dev/null
docker rmi -f $(docker images -q --filter "reference=dev-peer*") 2>/dev/null
```

---

## 七、复现图 17：性能对比测试

### 7.1 系统启动时间对比测试（重复 10 次求均值）

**对应论文图片**：图 17 密码算法时间开销对比（及论文 5.2.1 节中启动时间对比）

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 创建启动时间测试脚本
cat > /tmp/startup_time_test.sh << 'SCRIPT_EOF'
#!/bin/bash
# Fabric-gm 联盟链测试网络启动时间测试
# 重复 10 次，求均值

FABRIC_GM_DIR="$GOPATH/src/github.com/ponyletter/fabric-gm"
RESULT_FILE="/tmp/startup_time_results.txt"
NUM_TESTS=10

echo "=== Fabric-gm 启动时间测试（共 ${NUM_TESTS} 次）===" | tee $RESULT_FILE

cd $FABRIC_GM_DIR

total_time=0

for i in $(seq 1 $NUM_TESTS); do
    echo "--- 第 $i 次测试 ---" | tee -a $RESULT_FILE

    # 清理上一次的环境
    ./byfn.sh down > /dev/null 2>&1
    sleep 3

    # 记录开始时间
    START_TIME=$(date +%s%N)

    # 启动测试网络（-a 包含 CA）
    ./byfn.sh up -a > /tmp/byfn_test_${i}.log 2>&1

    # 记录结束时间
    END_TIME=$(date +%s%N)

    # 计算耗时（秒，保留 1 位小数）
    ELAPSED=$(echo "scale=1; ($END_TIME - $START_TIME) / 1000000000" | bc)
    echo "第 $i 次启动时间: ${ELAPSED} s" | tee -a $RESULT_FILE

    total_time=$(echo "$total_time + $ELAPSED" | bc)

    # 关闭网络
    ./byfn.sh down > /dev/null 2>&1
    sleep 3
done

# 计算均值
avg_time=$(echo "scale=1; $total_time / $NUM_TESTS" | bc)
echo "" | tee -a $RESULT_FILE
echo "=== 测试结果汇总 ===" | tee -a $RESULT_FILE
echo "总计测试次数: $NUM_TESTS" | tee -a $RESULT_FILE
echo "总耗时: ${total_time} s" | tee -a $RESULT_FILE
echo "平均启动时间: ${avg_time} s" | tee -a $RESULT_FILE
echo "（论文参考值: Fabric-gm 109.6s, 原生 Fabric 106.1s）" | tee -a $RESULT_FILE
SCRIPT_EOF

chmod +x /tmp/startup_time_test.sh

# 运行测试（预计总耗时约 30-40 分钟）
# bash /tmp/startup_time_test.sh
echo "启动时间测试脚本已创建: /tmp/startup_time_test.sh"
echo "执行命令: bash /tmp/startup_time_test.sh"
```

### 7.2 密码算法时间开销对比测试（重复 1000 次求均值）

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 创建密码算法性能测试脚本
cat > /tmp/crypto_benchmark_test.go << 'GOEOF'
package main

import (
    "crypto/ecdsa"
    "crypto/elliptic"
    "crypto/rand"
    "crypto/sha256"
    "fmt"
    "time"

    "github.com/tjfoc/gmsm/sm2"
    "github.com/tjfoc/gmsm/sm3"
)

func main() {
    numTests := 1000
    testData := []byte("This is a test message for performance benchmarking of cryptographic algorithms in Fabric-gm platform.")

    fmt.Println("=== 密码算法时间开销对比测试（各 1000 次求均值）===")
    fmt.Println()

    // ========== SM2 vs ECDSA 签名验签测试 ==========

    // SM2 密钥生成
    sm2PrivKey, _ := sm2.GenerateKey()

    // ECDSA 密钥生成（P256 曲线）
    ecdsaPrivKey, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)

    // SM2 签名性能测试
    var sm2SignTotal time.Duration
    for i := 0; i < numTests; i++ {
        start := time.Now()
        sm2PrivKey.Sign(rand.Reader, testData, nil)
        sm2SignTotal += time.Since(start)
    }
    fmt.Printf("SM2 签名平均时间:   %.3f ms\n", float64(sm2SignTotal.Microseconds())/float64(numTests)/1000.0)

    // SM2 验签性能测试
    sm2Sig, _ := sm2PrivKey.Sign(rand.Reader, testData, nil)
    var sm2VerifyTotal time.Duration
    for i := 0; i < numTests; i++ {
        start := time.Now()
        sm2PrivKey.PublicKey.Verify(testData, sm2Sig)
        sm2VerifyTotal += time.Since(start)
    }
    fmt.Printf("SM2 验签平均时间:   %.3f ms\n", float64(sm2VerifyTotal.Microseconds())/float64(numTests)/1000.0)

    // ECDSA 签名性能测试
    var ecdsaSignTotal time.Duration
    for i := 0; i < numTests; i++ {
        start := time.Now()
        ecdsa.Sign(rand.Reader, ecdsaPrivKey, testData[:32])
        ecdsaSignTotal += time.Since(start)
    }
    fmt.Printf("ECDSA 签名平均时间: %.3f ms\n", float64(ecdsaSignTotal.Microseconds())/float64(numTests)/1000.0)

    // ECDSA 验签性能测试
    r, s, _ := ecdsa.Sign(rand.Reader, ecdsaPrivKey, testData[:32])
    var ecdsaVerifyTotal time.Duration
    for i := 0; i < numTests; i++ {
        start := time.Now()
        ecdsa.Verify(&ecdsaPrivKey.PublicKey, testData[:32], r, s)
        ecdsaVerifyTotal += time.Since(start)
    }
    fmt.Printf("ECDSA 验签平均时间: %.3f ms\n", float64(ecdsaVerifyTotal.Microseconds())/float64(numTests)/1000.0)

    fmt.Println()

    // ========== SM3 vs SHA256 哈希测试 ==========

    dataSizes := []int{32, 64, 128, 256, 512, 1024, 2048, 4096}

    fmt.Println("--- SM3 vs SHA256 不同数据规模哈希时间对比 ---")
    fmt.Printf("%-12s %-15s %-15s\n", "数据大小", "SM3 (ms)", "SHA256 (ms)")

    for _, size := range dataSizes {
        data := make([]byte, size)
        rand.Read(data)

        // SM3 哈希测试
        var sm3Total time.Duration
        for i := 0; i < numTests; i++ {
            start := time.Now()
            h := sm3.New()
            h.Write(data)
            h.Sum(nil)
            sm3Total += time.Since(start)
        }

        // SHA256 哈希测试
        var sha256Total time.Duration
        for i := 0; i < numTests; i++ {
            start := time.Now()
            h := sha256.New()
            h.Write(data)
            h.Sum(nil)
            sha256Total += time.Since(start)
        }

        sm3Avg := float64(sm3Total.Microseconds()) / float64(numTests) / 1000.0
        sha256Avg := float64(sha256Total.Microseconds()) / float64(numTests) / 1000.0

        fmt.Printf("%-12d %-15.6f %-15.6f\n", size, sm3Avg, sha256Avg)
    }

    fmt.Println()
    fmt.Println("=== 测试完成 ===")
    fmt.Println("（对比论文图 17 中的时间开销数据）")
}
GOEOF

echo "性能测试脚本已创建: /tmp/crypto_benchmark_test.go"
echo "将此文件放到 fabric-gm 项目中运行，或使用 go test -bench 方式运行"
```

### 7.3 使用 Go Benchmark 进行标准性能测试

```bash
cd $GOPATH/src/github.com/ponyletter/fabric-gm

# 创建标准 Go Benchmark 测试文件
cat > /tmp/crypto_bench_test.go << 'GOEOF'
package gm_test

import (
    "crypto/ecdsa"
    "crypto/elliptic"
    "crypto/rand"
    "crypto/sha256"
    "testing"

    "github.com/tjfoc/gmsm/sm2"
    "github.com/tjfoc/gmsm/sm3"
)

var testMsg = []byte("Benchmark test message for Fabric-gm cryptographic performance analysis")

// SM2 签名 Benchmark
func BenchmarkSM2Sign(b *testing.B) {
    privKey, _ := sm2.GenerateKey()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        privKey.Sign(rand.Reader, testMsg, nil)
    }
}

// SM2 验签 Benchmark
func BenchmarkSM2Verify(b *testing.B) {
    privKey, _ := sm2.GenerateKey()
    sig, _ := privKey.Sign(rand.Reader, testMsg, nil)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        privKey.PublicKey.Verify(testMsg, sig)
    }
}

// ECDSA 签名 Benchmark
func BenchmarkECDSASign(b *testing.B) {
    privKey, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
    digest := sha256.Sum256(testMsg)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        ecdsa.Sign(rand.Reader, privKey, digest[:])
    }
}

// ECDSA 验签 Benchmark
func BenchmarkECDSAVerify(b *testing.B) {
    privKey, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
    digest := sha256.Sum256(testMsg)
    r, s, _ := ecdsa.Sign(rand.Reader, privKey, digest[:])
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        ecdsa.Verify(&privKey.PublicKey, digest[:], r, s)
    }
}

// SM3 哈希 Benchmark
func BenchmarkSM3Hash(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        h := sm3.New()
        h.Write(testMsg)
        h.Sum(nil)
    }
}

// SHA256 哈希 Benchmark
func BenchmarkSHA256Hash(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        h := sha256.New()
        h.Write(testMsg)
        h.Sum(nil)
    }
}
GOEOF

# 将测试文件复制到 bccsp/gm 目录下运行
cp /tmp/crypto_bench_test.go \
    $GOPATH/src/github.com/ponyletter/fabric-gm/bccsp/gm/crypto_bench_test.go

# 运行 Benchmark 测试
cd $GOPATH/src/github.com/ponyletter/fabric-gm
go test -bench=. -benchmem -count=3 ./bccsp/gm/... 2>&1 | tee /tmp/benchmark_results.log

# 查看结果
cat /tmp/benchmark_results.log

# 清理
# rm $GOPATH/src/github.com/ponyletter/fabric-gm/bccsp/gm/crypto_bench_test.go
```

### 7.4 交易时间与动态证书生成时间测试

```bash
# 创建交易时间测试脚本
cat > /tmp/transaction_time_test.sh << 'SCRIPT_EOF'
#!/bin/bash
# 交易时间与证书生成时间测试
# 每项测试 1000 次，求均值

echo "=== 交易时间开销测试（1000 次求均值）==="

# 前提：Fabric-gm 测试网络已启动（./byfn.sh up -a）

NUM_TESTS=1000
TOTAL_TX_TIME=0

# 交易时间测试
for i in $(seq 1 $NUM_TESTS); do
    START=$(date +%s%N)

    # 通过 CLI 容器执行链码调用（invoke）
    docker exec cli peer chaincode invoke \
        -o orderer.example.com:7050 \
        -C mychannel \
        -n mycc \
        -c '{"Args":["invoke","a","b","10"]}' \
        --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
        > /dev/null 2>&1

    END=$(date +%s%N)
    ELAPSED=$(echo "scale=3; ($END - $START) / 1000000" | bc)
    TOTAL_TX_TIME=$(echo "$TOTAL_TX_TIME + $ELAPSED" | bc)

    # 每 100 次输出进度
    if [ $((i % 100)) -eq 0 ]; then
        echo "已完成 $i / $NUM_TESTS 次交易测试"
    fi
done

AVG_TX_TIME=$(echo "scale=3; $TOTAL_TX_TIME / $NUM_TESTS" | bc)
echo "平均交易时间: ${AVG_TX_TIME} ms"
echo "（论文参考值: Fabric-gm 53.741ms, 原生 Fabric 23.534ms）"

echo ""
echo "=== 动态证书生成时间测试（1000 次求均值）==="

TOTAL_CERT_TIME=0

for i in $(seq 1 $NUM_TESTS); do
    USER_NAME="testuser${i}"

    START=$(date +%s%N)

    # 注册用户
    docker exec ca_org1 fabric-ca-client register \
        --id.name $USER_NAME \
        --id.secret ${USER_NAME}pw \
        --id.type client \
        -u http://localhost:7054 \
        > /dev/null 2>&1

    # 生成证书（enroll）
    docker exec ca_org1 fabric-ca-client enroll \
        -u http://${USER_NAME}:${USER_NAME}pw@localhost:7054 \
        -M /tmp/${USER_NAME}/msp \
        > /dev/null 2>&1

    END=$(date +%s%N)
    ELAPSED=$(echo "scale=3; ($END - $START) / 1000000" | bc)
    TOTAL_CERT_TIME=$(echo "$TOTAL_CERT_TIME + $ELAPSED" | bc)

    if [ $((i % 100)) -eq 0 ]; then
        echo "已完成 $i / $NUM_TESTS 次证书生成测试"
    fi
done

AVG_CERT_TIME=$(echo "scale=3; $TOTAL_CERT_TIME / $NUM_TESTS" | bc)
echo "平均证书生成时间: ${AVG_CERT_TIME} ms"
echo "（论文参考值: 管理员 114.838ms, 普通用户 199.285ms）"

echo ""
echo "=== 测试完成 ==="
SCRIPT_EOF

chmod +x /tmp/transaction_time_test.sh

echo "交易时间测试脚本已创建: /tmp/transaction_time_test.sh"
echo "使用前请确保 Fabric-gm 测试网络已启动（./byfn.sh up -a）"
echo "执行命令: bash /tmp/transaction_time_test.sh"
```

---

## 八、实验结果与论文图片对应关系总结

| 论文图片 | 实验内容 | 复现命令/脚本 | 执行目录 |
|---------|---------|-------------|---------|
| 图 10 | SM3 哈希接口调用可用性验证 | `go test -v ./bccsp/gm/... -run TestSM3` | fabric-gm 根目录 |
| 图 11 | SM3 哈希接口有效性验证 | `gmssl sm3` + `go test -v ./bccsp/gm/... -run TestHash` | fabric-gm 根目录 |
| 图 12 | SM4 算法接口可用性和有效性验证 | `go test -v ./bccsp/gm/... -run TestSM4` | fabric-gm 根目录 |
| 图 13 | SM2 算法接口可用性和有效性验证 | `go test -v ./bccsp/gm/... -run TestSM2` | fabric-gm 根目录 |
| 图 14 | cryptogen 支持国密证书生成 | `make cryptogen` + `./build/bin/cryptogen generate` | fabric-gm 根目录 |
| 图 15 | Fabric-CA-gm 支持国密证书生成 | `fabric-ca-client register/enroll` + `gmssl x509` | CA 容器内/外 |
| 图 16 | 联盟链测试网络正常启动 | `./byfn.sh up -a` | fabric-gm 根目录 |
| 图 17 | 密码算法时间开销对比 | `go test -bench=.` + 启动时间测试脚本 | fabric-gm 根目录 |

---

## 九、常见问题排查

### 9.1 编译错误排查

```bash
# 问题 1：go module 相关错误
# 解决：确保在正确的 GOPATH 路径下，且 vendor 目录完整
cd $GOPATH/src/github.com/ponyletter/fabric-gm
ls vendor/
go mod verify

# 问题 2：缺少依赖包
# 解决：使用 go mod vendor 重新下载
go mod vendor

# 问题 3：CGO 编译错误
# 解决：安装 gcc 和相关开发库
sudo apt-get install -y gcc g++ libc6-dev

# 问题 4：Docker 权限问题
# 解决：确保用户在 docker 组
sudo usermod -aG docker $USER
newgrp docker
```

### 9.2 测试网络启动失败

```bash
# 清理所有残留容器和网络
docker rm -f $(docker ps -aq) 2>/dev/null
docker network prune -f
docker volume prune -f

# 重新构建镜像
cd $GOPATH/src/github.com/ponyletter/fabric-gm
make docker

# 重试启动
./byfn.sh down
./byfn.sh up -a
```

### 9.3 证书验证失败

```bash
# 如果 GmSSL 无法解析证书，可能是版本兼容性问题
# 尝试使用 openssl 查看基本信息
openssl x509 -in cert.pem -text -noout

# 或者使用 Go 程序解析
cat > /tmp/parse_cert.go << 'EOF'
package main

import (
    "crypto/x509"
    "encoding/pem"
    "fmt"
    "io/ioutil"
    "os"
)

func main() {
    certFile := os.Args[1]
    certPEM, _ := ioutil.ReadFile(certFile)
    block, _ := pem.Decode(certPEM)
    cert, err := x509.ParseCertificate(block.Bytes)
    if err != nil {
        fmt.Printf("标准 x509 解析失败（可能是国密证书）: %v\n", err)
        fmt.Println("证书原始字节已读取成功，这说明文件格式正确")
        return
    }
    fmt.Printf("签名算法: %s\n", cert.SignatureAlgorithm)
    fmt.Printf("公钥算法: %s\n", cert.PublicKeyAlgorithm)
    fmt.Printf("使用者: %s\n", cert.Subject)
    fmt.Printf("颁发者: %s\n", cert.Issuer)
}
EOF
go run /tmp/parse_cert.go /path/to/cert.pem
```

---

## 十、附录：关键源码文件路径速查

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| BCCSP 国密实例主体 | `bccsp/gm/impl.go` | 国密 BCCSP 实例化与接口注册 |
| SM2 签名验签 | `bccsp/gm/sm2.go` | signGMSM2、verifyGMSM2 |
| SM2 密钥定义 | `bccsp/gm/sm2key.go` | gmsm2PrivateKey、gmsm2PublicKey |
| SM4 加解密 | `bccsp/gm/sm4.go` | SM4Encrypt、SM4Decrypt |
| SM4 密钥定义 | `bccsp/gm/sm4key.go` | gmsm4PrivateKey |
| 密钥生成 | `bccsp/gm/keygen.go` | gmsm2KeyGenerator、gmsm4KeyGenerator |
| 国密证书辅助 | `bccsp/gm/certhelper.go` | 国密证书生成与转换 |
| 工厂默认配置 | `bccsp/factory/opts.go` | ProviderName: "GM", HashFamily: "GMSM3" |
| 底层 SM2 实现 | `tjfoc/gmsm/sm2/` | 同济开源 SM2 算法实现 |
| 底层 SM3 实现 | `tjfoc/gmsm/sm3/` | 同济开源 SM3 算法实现 |
| 底层 SM4 实现 | `tjfoc/gmsm/sm4/` | 同济开源 SM4 算法实现 |
| cryptogen 入口 | `cmd/cryptogen/main.go` | 证书生成工具 |
| Makefile | `Makefile` | 编译目标定义 |
| go.mod | `go.mod` | 模块依赖（module: github.com/ponyletter/fabric-gm） |
