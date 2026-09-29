# CIP 规范文件

- `for-ai.md`：**给 AI 编程助手的接入指南。** 开发者只需对自己的 AI 说「读这个文件，按步骤检查并改造我的项目」。
- `companion.schema.json`：`companion.json` 的 JSON Schema（Draft 2020-12）。协议正文见 [PROTOCOL.zh-CN.md](../PROTOCOL.zh-CN.md)。
- `examples/`：按真实仓库结构写的示例，未经各项目作者确认，仅作格式参考。

校验一个描述文件：

```bash
pip install jsonschema
python -c "import json,sys,jsonschema; jsonschema.validate(json.load(open(sys.argv[1])), json.load(open('spec/companion.schema.json'))); print('OK')" companion.json
```
