---
title: Dependencies Increase Application Startup Time
tags: [til, blazor, webassembly, initialization, performance]
readtime: true
---

Dependencies like Telerik UI for Blazor & AutoMapper increase development velocity. However, they increase the application startup time. I created sample applications to illustrate the difference:

| Device / Load Time (ms) | [Without AutoMapper](https://zhiyuan-amos.github.io/BlazorWasmBarebone/) ([source code](https://github.com/Zhiyuan-Amos/BlazorWasmBarebone)) | [With AutoMapper](https://zhiyuan-amos.github.io/BlazorWasmAutoMapper/) ([source code](https://github.com/Zhiyuan-Amos/BlazorWasmAutoMapper)) |
|-|-|-|
| MacBook Pro 2015 | 200 | 350 |
| Samsung Galaxy A31 | 600 | 1800 |

The increase in startup time is pretty drastic on devices with limited processing power, so if you are creating a Blazor WASM application for external usage where you have no control over what devices users use to access your application, then you should consider removing unnecessary dependencies from your application.
