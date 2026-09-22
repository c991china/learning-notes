# Python 异步要点

- `async def` 定义协程，`await` 等待可等待对象。
- 用 `asyncio.run(main())` 启动事件循环。
- I/O 密集型（网络/文件）才适合异步，CPU 密集用多进程。
- `asyncio.gather(*tasks)` 并发运行多个协程。
- 避免在协程里调用阻塞函数（如 `time.sleep`），改用 `await asyncio.sleep`。
