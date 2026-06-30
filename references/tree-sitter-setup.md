# tree-sitter 安装(Tier 2)

Tier 2 依赖 tree-sitter **CLI** + 各语言 **grammar**。注意 brew 有坑(见下)。装好才能产出语法级调用图/依赖图 + 精确 cite 行号。

## 第一步:装 CLI(注意 brew 坑)

⚠️ `brew install tree-sitter` 装的是**库**(libtree-sitter),不是 CLI——命令行 `tree-sitter` 不会有。brew caveats 明确写了:要 CLI 用另一个 formula。

```bash
brew install tree-sitter-cli        # macOS,装的是 CLI(正确)
# 其他方式:
# npm install -g tree-sitter-cli
# cargo install tree-sitter-cli
```

验证:`tree-sitter --version`(应输出版本号,如 0.26.10)。没输出 = 装错了(装成库了)。

## 第二步:配 grammar(CLI 装了还不够)

tree-sitter CLI 解析某语言需要对应 grammar。机制:`tree-sitter init-config` 生成 `~/.config/tree-sitter/config.json`,里面有 `parser-directories` 列表;tree-sitter 扫描这些目录下的 `tree-sitter-*` 仓库找 grammar。**只装 CLI 不配 grammar,parse 会报 `No language found`。**

配 Go grammar(实测可用):
```bash
tree-sitter init-config                                    # 生成 config.json
mkdir -p ~/github                                          # config 默认 parser-directories 之一
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-go ~/github/tree-sitter-go
tree-sitter parse <任意.go文件>                            # 验证,应输出 AST(带 [row,col])
```

其他语言 grammar 同理 clone 到 `parser-directories` 所列目录(tree-sitter-python / tree-sitter-typescript / tree-sitter-solidity 等),按目标仓库实际技术栈装。taskon 系列以 Go 为主,前端 Vue(taskon-website),合约 Solidity / Move。

`parser-directories` 默认含 `~/github`、`~/src`、`~/source`、`~/projects`、`~/dev`、`~/git`;可编辑 config.json 自加目录。

## 装不上怎么办(努力原则)

**先努力装,确认装不上才兜底。** tree-sitter-cli + grammar 通常一两分钟能装好(brew bottle + git clone),收益(语法级调用图、精确 cite 行号)远大于成本。

探测失败时,如实告知并给升级路径:
> 当前 Tier 3(grep 兜底),调用图靠推断。装 tree-sitter 可升级 Tier 2,质量显著提升:
> `brew install tree-sitter-cli` + clone grammar(见 `references/tree-sitter-setup.md`)
> 要我帮你装吗?

请用户协助配置,而不是静默降级——用户不知道你在兜底,会误信不可靠的调用图。
