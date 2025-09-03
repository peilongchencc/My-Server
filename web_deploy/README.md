# 网页零停机部署原理:

## 效果:

使用的是蓝绿部署策略，核心思想是通过符号链接实现原子性切换，让用户在网页更新过程中完全无感知。

## 核心代码:

创建 `deploy.sh` 文件，写入以下代码:

```bash
#!/bin/bash

# 零停机部署脚本
# 使用蓝绿部署策略，通过符号链接实现原子性切换

set -e  # 遇到错误时退出

DEPLOY_DIR="/project/front"
APP_NAME="interview"
BACKUP_COUNT=3  # 保留的历史版本数量

# 颜色输出函数
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# 检查必要文件
check_prerequisites() {
    log_info "检查部署前提条件..."
    
    if [ ! -f "${DEPLOY_DIR}/dist.zip" ]; then
        log_error "dist.zip 文件不存在，请先上传新版本文件"
        exit 1
    fi
    
    log_info "前提条件检查通过"
}

# 初始化目录结构（仅在首次部署时需要）
init_deployment() {
    log_info "初始化部署结构..."
    
    cd "$DEPLOY_DIR"
    
    # 如果interview是普通目录，需要先备份并转换为符号链接结构
    if [ -d "$APP_NAME" ] && [ ! -L "$APP_NAME" ]; then
        log_warn "检测到现有的interview目录，正在转换为蓝绿部署模式..."
        
        # 创建时间戳版本号
        TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
        CURRENT_VERSION="${APP_NAME}_${TIMESTAMP}"
        
        # 移动当前版本到版本化目录
        mv "$APP_NAME" "$CURRENT_VERSION"
        
        # 创建符号链接指向当前版本
        ln -sf "$CURRENT_VERSION" "$APP_NAME"
        
        log_info "已转换为蓝绿部署模式，当前版本: $CURRENT_VERSION"
    fi
}

# 部署新版本
deploy_new_version() {
    log_info "开始部署新版本..."
    
    cd "$DEPLOY_DIR"
    
    # 生成新版本目录名
    TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
    NEW_VERSION="${APP_NAME}_${TIMESTAMP}"
    
    log_info "新版本目录: $NEW_VERSION"
    
    # 解压新版本到临时目录
    log_info "解压新版本文件..."
    unzip -q dist.zip
    
    # 重命名为版本化目录
    mv dist "$NEW_VERSION"
    
    # 验证新版本
    if [ ! -f "${NEW_VERSION}/index.html" ]; then
        log_error "新版本验证失败：缺少index.html文件"
        rm -rf "$NEW_VERSION"
        exit 1
    fi
    
    log_info "新版本准备完成: $NEW_VERSION"
    
    # 原子性切换
    log_info "执行原子性切换..."
    CURRENT_LINK=$(readlink "$APP_NAME" 2>/dev/null || echo "")
    
    # 确保在正确的目录下执行切换操作
    cd "$DEPLOY_DIR"
    
    # 真正的原子性符号链接切换
    # 使用ln的-T选项确保原子性替换，-f强制覆盖
    ln -Tsfn "$NEW_VERSION" "$APP_NAME"
    
    # 验证符号链接切换是否成功
    ACTUAL_LINK=$(readlink "${DEPLOY_DIR}/${APP_NAME}" 2>/dev/null || echo "")
    if [ "$ACTUAL_LINK" = "$NEW_VERSION" ]; then
        log_info "切换完成！新版本已上线"
    else
        log_error "符号链接切换失败！期望: $NEW_VERSION，实际: $ACTUAL_LINK"
        # 清理失败的新版本
        rm -rf "$NEW_VERSION"
        exit 1
    fi
    
    # 记录当前和上一个版本
    echo "$NEW_VERSION" > "${APP_NAME}.current"
    if [ -n "$CURRENT_LINK" ]; then
        echo "$CURRENT_LINK" > "${APP_NAME}.previous"
    fi
    
    return 0
}

# 测试nginx配置并重载
reload_nginx() {
    log_info "测试nginx配置..."
    
    if sudo nginx -t; then
        log_info "nginx配置测试通过，重载nginx..."
        sudo nginx -s reload
        log_info "nginx重载完成"
    else
        log_error "nginx配置测试失败！"
        return 1
    fi
}

# 清理旧版本
cleanup_old_versions() {
    log_info "清理旧版本..."
    
    cd "$DEPLOY_DIR"
    
    # 获取当前版本（优先从记录文件读取）
    if [ -f "${APP_NAME}.current" ]; then
        CURRENT_VERSION=$(cat "${APP_NAME}.current")
    else
        CURRENT_VERSION=$(readlink "$APP_NAME" 2>/dev/null || echo "")
    fi
    
    if [ -z "$CURRENT_VERSION" ]; then
        log_warn "无法获取当前版本，跳过清理"
        return 0
    fi
    
    # 获取所有版本目录并按版本名排序（保留最新的几个版本）
    VERSIONS=$(ls -d ${APP_NAME}_* 2>/dev/null | grep -v "\.current\|\.previous" | sort -r || true)
    
    if [ -z "$VERSIONS" ]; then
        log_info "没有找到旧版本"
        return 0
    fi
    
    # 转换为数组并保留最新的几个版本
    VERSION_ARRAY=($VERSIONS)
    TOTAL_VERSIONS=${#VERSION_ARRAY[@]}
    
    if [ $TOTAL_VERSIONS -gt $BACKUP_COUNT ]; then
        log_info "保留最新 $BACKUP_COUNT 个版本，删除其余 $((TOTAL_VERSIONS - BACKUP_COUNT)) 个旧版本"
        
        for (( i=$BACKUP_COUNT; i<$TOTAL_VERSIONS; i++ )); do
            OLD_VERSION="${VERSION_ARRAY[$i]}"
            if [ "$OLD_VERSION" != "$CURRENT_VERSION" ]; then
                log_info "删除旧版本: $OLD_VERSION"
                rm -rf "$OLD_VERSION"
            fi
        done
    else
        log_info "当前版本数量 ($TOTAL_VERSIONS) 未超过保留数量 ($BACKUP_COUNT)，不需要清理"
    fi
}

# 回滚到上一个版本
rollback() {
    log_info "开始回滚到上一个版本..."
    
    cd "$DEPLOY_DIR"
    
    if [ ! -f "${APP_NAME}.previous" ]; then
        log_error "没有找到上一个版本信息，无法回滚"
        exit 1
    fi
    
    PREVIOUS_VERSION=$(cat "${APP_NAME}.previous")
    
    if [ ! -d "$PREVIOUS_VERSION" ]; then
        log_error "上一个版本目录不存在: $PREVIOUS_VERSION"
        exit 1
    fi
    
    # 保存当前版本作为previous
    CURRENT_VERSION=$(readlink "$APP_NAME" 2>/dev/null || echo "")
    
    # 真正的原子性符号链接切换到上一个版本
    # 使用ln的-T选项确保原子性替换，-f强制覆盖
    ln -Tsfn "$PREVIOUS_VERSION" "$APP_NAME"
    
    # 验证回滚是否成功
    ACTUAL_LINK=$(readlink "$APP_NAME" 2>/dev/null || echo "")
    if [ "$ACTUAL_LINK" = "$PREVIOUS_VERSION" ]; then
        log_info "回滚完成！当前版本: $PREVIOUS_VERSION"
    else
        log_error "回滚失败！期望: $PREVIOUS_VERSION，实际: $ACTUAL_LINK"
        exit 1
    fi
    
    # 更新版本记录
    echo "$PREVIOUS_VERSION" > "${APP_NAME}.current"
    if [ -n "$CURRENT_VERSION" ]; then
        echo "$CURRENT_VERSION" > "${APP_NAME}.previous"
    fi
    
    # 重载nginx
    reload_nginx
}

# 显示当前状态
show_status() {
    cd "$DEPLOY_DIR"
    
    echo "==================== 部署状态 ===================="
    
    if [ -L "$APP_NAME" ]; then
        CURRENT_VERSION=$(readlink "$APP_NAME")
        echo "当前版本: $CURRENT_VERSION"
    else
        echo "当前版本: 未使用符号链接模式"
    fi
    
    if [ -f "${APP_NAME}.previous" ]; then
        PREVIOUS_VERSION=$(cat "${APP_NAME}.previous")
        echo "上一版本: $PREVIOUS_VERSION"
    else
        echo "上一版本: 无"
    fi
    
    echo
    echo "所有版本目录:"
    ls -la ${APP_NAME}_* 2>/dev/null | grep "^d" || echo "  无版本目录"
    
    echo "================================================"
}

# 主函数
main() {
    case "${1:-deploy}" in
        "deploy")
            log_info "开始零停机部署..."
            check_prerequisites
            init_deployment
            deploy_new_version
            reload_nginx
            cleanup_old_versions
            log_info "部署完成！"
            show_status
            ;;
        "rollback")
            rollback
            ;;
        "status")
            show_status
            ;;
        "cleanup")
            cleanup_old_versions
            ;;
        *)
            echo "用法: $0 {deploy|rollback|status|cleanup}"
            echo "  deploy  - 部署新版本（默认）"
            echo "  rollback - 回滚到上一个版本"
            echo "  status  - 显示当前部署状态"
            echo "  cleanup - 清理旧版本"
            exit 1
            ;;
    esac
}

# 执行主函数
main "$@"
```

## 操作步骤

### 📋 日常部署流程

**1. 接收新版本文件**
```bash
# 前端同事会给你一个 dist.zip 文件
# 将它上传到服务器的 /project/front 目录
```

**2. 执行部署**
```bash
# 进入部署目录
cd /project/front

# 给脚本执行权限（只需要第一次执行）
chmod +x deploy.sh

# 执行部署
./deploy.sh deploy
```

**3. 验证部署结果**
```bash
# 检查部署状态
./deploy.sh status

# 检查网站是否正常访问
curl -I https://test.interview.aistar.com
```

### 🔄 紧急回滚

如果发现新版本有问题：
```bash
# 立即回滚到上一个版本
./deploy.sh rollback
```

### 🧹 定期维护

```bash
# 清理旧版本（保留最新3个版本）
./deploy.sh cleanup

# 监控访问日志
sudo tail -f /var/log/nginx/interview_access.log

# 监控错误日志  
sudo tail -f /var/log/nginx/interview_error.log
```

## 安全特性

### 1. 验证和回滚机制
- 脚本会验证新版本是否包含 `index.html`
- 自动保留上一个版本信息，支持一键回滚
- 保留最新3个历史版本作为备份

### 2. Nginx安全配置
从你提供的nginx配置可以看到：
- SSL/TLS加密
- 防止PHP文件执行
- HTTP方法限制
- 静态文件缓存优化

## 目录结构示例

部署后的目录结构会是这样：
```
/project/front/
├── interview -> interview_20240115_153045  # 符号链接，指向当前版本
├── interview_20240115_153045/              # 当前版本
├── interview_20240115_143022/              # 上一个版本  
├── interview_20240115_133011/              # 更早版本
├── interview.current                       # 记录当前版本名
├── interview.previous                      # 记录上一版本名
├── dist.zip                               # 待部署的新版本文件
└── deploy.sh                              # 部署脚本
```

## 关键优势

1. **零停机时间**：符号链接切换是原子操作，用户无感知
2. **快速回滚**：出问题时可以秒级回滚到上一版本
3. **版本管理**：自动保留历史版本，便于追溯和回滚
4. **安全验证**：部署前验证文件完整性
5. **自动清理**：避免磁盘空间浪费

## 版本命名意义:

版本号对应的是具体的时间，例如 `interview_20250829_141222` 代表 2025-08-29 14:12:22 执行的方案。

执行效果如下:

```bash
(base) root@iZ2ze50qtwycx9cbb:/project/front# date +"%Y%m%d_%H%M%S"
20250901_132511
(base) root@iZ2ze50qtwycx9cbb:/project/front# 
```

这样设计的好处:

1. 唯一性：精确到秒，避免同名冲突
2. 可读性：一眼就能看出部署时间
3. 排序性：按字母顺序排序就是按时间顺序
4. 追溯性：方便查找特定时间的部署版本


## 注意事项

⚠️ **重要提醒**：
- 确保 `dist.zip` 文件放在 `/project/front` 目录下
- 部署前可以先执行 `./deploy.sh status` 查看当前状态
- 如果出现问题，立即执行 `./deploy.sh rollback`
- 定期执行 `./deploy.sh cleanup` 清理旧版本

这个方案非常成熟和安全，即使是新手也可以放心使用。有任何问题随时问我！
