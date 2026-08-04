# FF14 石之家签到脚本

> [!NOTE]
> **与上游仓库的差异说明：**
> 
> 1. **添加了自动重试与多账户支持**：
>    在 `.github/workflows/daily.yml` 中增加了失败重试逻辑（间隔 5 分钟重试，最多 3 次），并支持通过 Secrets 配置多账户循环签到（最多支持 5 个账号）。每次重试使用全新容器（`docker run`），保证运行环境干净。
> 2. **移除了源码文件**：
>    删除了除 `README.md` 和 `.github/workflows/daily.yml` 之外的开发与源码文件。运行工作流时直接拉取上游编译好的 Docker 镜像。

## 使用方法

- Fork 本仓库或者通过手动在仓库中新建 `.github/workflows/daily.yml` 文件，内容如下：

```yaml
name: Daily Tasks

on:
  schedule:
    # Runs at 10:30 am UTC+8 every day
    - cron: "30 2 * * *"
  workflow_dispatch:

jobs:
  run-tasks:
    name: Run FF14 Risingstone Tasks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Run FF14 Risingstone Tasks with Docker Restart Retry
        env:
          COOKIE_1: ${{ secrets.COOKIE }}
          USER_AGENT_1: ${{ secrets.USER_AGENT }}
          COOKIE_2: ${{ secrets.COOKIE_2 }}
          USER_AGENT_2: ${{ secrets.USER_AGENT_2 }}
          COOKIE_3: ${{ secrets.COOKIE_3 }}
          USER_AGENT_3: ${{ secrets.USER_AGENT_3 }}
          COOKIE_4: ${{ secrets.COOKIE_4 }}
          USER_AGENT_4: ${{ secrets.USER_AGENT_4 }}
          COOKIE_5: ${{ secrets.COOKIE_5 }}
          USER_AGENT_5: ${{ secrets.USER_AGENT_5 }}
        run: |
          max_attempts=3
          retry_wait=300
          overall_exit=0

          for idx in $(seq 1 5); do
            cookie_var="COOKIE_$idx"
            ua_var="USER_AGENT_$idx"
            cookie="${!cookie_var}"
            ua="${!ua_var}"

            if [ -z "$cookie" ]; then
              continue
            fi

            if [ -z "$ua" ]; then
              ua="$USER_AGENT_1"
            fi

            echo "=========================================="
            echo "Processing Account $idx..."
            echo "=========================================="

            account_status=0
            for i in $(seq 1 $max_attempts); do
              echo "Account $idx - Attempt $i of $max_attempts..."

              # 每次重试前清理旧容器，保证环境干净
              docker rm -f risingstone_task || true

              status=0
              docker run --name risingstone_task \
                -e INPUT_COOKIE="$cookie" \
                -e INPUT_USER_AGENT="$ua" \
                ghcr.io/starhearthunt/ff14risingstone_sign_task:master || status=$?

              if [ $status -eq 0 ]; then
                echo "Account $idx - Success on attempt $i!"
                account_status=0
                break
              fi

              account_status=$status
              if [ $i -lt $max_attempts ]; then
                echo "Account $idx - Attempt $i failed (exit code: $status). Waiting ${retry_wait}s before retrying..."
                sleep $retry_wait
              else
                echo "Account $idx - Attempt $i failed (exit code: $status). All attempts exhausted."
              fi
            done

            echo "Cleaning up container for Account $idx..."
            docker rm -f risingstone_task || true

            if [ $account_status -ne 0 ]; then
              echo "Account $idx failed execution."
              overall_exit=1
            fi
          done

          exit $overall_exit
```

- 在 Settings > Secrets and variables > Actions，添加对应的 Secret：

### 单账号设置

1. `COOKIE`

值为 `Cookie` 头中以等号 `=` 分割的 `ff14risingstones` 键值对，其中右值为 urlencode 后的结果。

例：

```bash
ff14risingstones=s%3A1111.2222222%2F33333
```

2. `USER_AGENT`

> [!NOTE]
> 由于石之家 API 新增的检测机制，需要设置与登录（获取 Cookie）时相同的 User-Agent 头，详情参考 [#17](https://github.com/StarHeartHunt/ff14risingstone_sign_task/issues/17)

值为登录时向石之家 API 所发送的 `User-Agent` 头。

例：

```bash
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36
```

### 多账号设置

如需为小号或其他账号添加自动签到，可以在 GitHub Secrets 中添加附加账号配置：

- `COOKIE_2`, `COOKIE_3`, `COOKIE_4`, `COOKIE_5`：对应小号的 `COOKIE` 凭据。
- `USER_AGENT_2`, `USER_AGENT_3`, `USER_AGENT_4`, `USER_AGENT_5`：（可选）对应账号使用的 User-Agent。若未配置，脚本将默认回退使用主账号的 `USER_AGENT`。

## 配置项

### 必填配置项

- `cookie`：石之家 API cookie 中的 `ff14risingstones` 键值对
- `user_agent`：请求 API 时使用的用户代理

### 可选配置项

可选配置项可以从 action 文件的 with段传入，和必填项一样。

- `base_url`：API 的入口域名。默认值：`https://apiff14risingstones.web.sdo.com`
- `comment_content`：完成评论任务时的评论内容。默认值：`<p><span class="at-emo">[emo6]</span>&nbsp;</p>`
- `like_post_id`：完成点赞任务时要点赞的根帖子 id。默认值：`8`
- `comment_post_id`：完成评论任务时要评论的根帖子 id。默认值：`8`
- `check_house_remain`：是否检查角色房屋拆除倒计时。默认值：`false`
- `get_sign_reward`：是否使用脚本领取当月签到奖励。默认值：`true`

## 许可证

本仓库使用 MIT 许可证

