# Research Tips

* When an experiment has been running for a long time without results: check the correctness of the code, run a small-scale experiment to estimate the time, or split it into multiple parallelizable experiments.
* When using PyTorch, utilize the MPS device to accelerate computations.
* Maintain a clean git repository: only commit essential content. Do not commit outdated versions, temporary test files, or intermediate log files.
* Complete tasks one by one. Add remaining tasks and items to be confirmed later to HEARTBEAT.md, clearly marking dependencies. Only when you are idle (or waiting for some tasks to finish), consider starting parallel tasks that have no dependencies and do not interfere with each other.

## Frontend Development Tips (Alpha Mining Project)

* API 引用问题：确保 JavaScript API 对象正确导出到 window 对象，且脚本加载顺序正确。api.js 必须在 backtest.js 和 portfolio.js 之前加载。
* 表单验证：使用 HTML5 内置验证时，浏览器默认错误消息不够清晰。应添加自定义验证函数，提供具体的错误提示（如："学习率必须在 0.001-1 之间"）。
* 日期控件样式：input[type="date"] 需要在 CSS 中单独指定样式，使其与其他输入框保持一致。
* 图标选择：优先使用 Font Awesome 图标而非 emoji，提供更专业、统一的外观。

## Backend API Testing Tips (Alpha Mining Project)

* Qlib instruments API：D.instruments(market='csi300') 在某些版本可能返回错误格式。备选方案是直接从 instruments 文件读取股票列表（如 ~/.qlib/qlib_data/cn_data/instruments/csi300.txt）。
* 数据库架构迁移：当数据库模型变更时，最简单的方法是删除旧数据库文件重新创建。长期应实现数据库迁移机制。
* API 参数设计：FastAPI 中，使用 Query() 声明 URL 查询参数，使用 Body() 或直接类型声明声明请求体参数。注意区分 GET 和 POST 请求的参数传递方式。
* 测试分离：直接测试业务逻辑（如 MyAlpha handler）和通过 API 测试是两个层面。直接测试可以快速定位问题，API 测试验证端到端功能。