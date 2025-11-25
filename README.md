ThinFAT32
=========

A lightweight implementation of the FAT32 filesystem for embedded systems.

Objective
---------
The purpose of ThinFAT32 is to be an easy to deploy, low-resource FAT32 filesystem for your embedded application.  The idea is not to go for speed or a lot of wacky features, but basic functionality in a library that is *easy to use.*  Let's not use a lot of super-optimized code, wacky linker techniques, or kooky C language features.  Let's just imlpement FAT32 by the books, as efficiently as we can without going apeshit, and then use it in our embedded systems.

-------
## HowToStart
```cmd
make clean && make all
./scripts/makefs make
./build/tests
```

## 原项目仓库
git@github.com:ryansturmer/thinfat32.git 

# FAT32 后端镜像文件指针与扇区读写函数改进说明

## 变更内容

本次更新对 FAT32 文件系统后端镜像管理及底层扇区读写接口做了如下重要改进：

### 1. 后端镜像文件管理接口

- 新增了全局文件指针 `g_backend`，用于管理当前挂载的 FAT32 镜像文件：

  ```c
  static FILE *g_backend = NULL;
  ```

- 新增 `tf_attach_image(const char *path)` 用于挂载（打开）指定的镜像文件。  
  打开前自动关闭可能已挂载的旧镜像，避免资源泄漏或冲突。  
  调用成功后，所有底层读写操作都通过这个全局指针进行。

- 新增 `tf_detach_image(void)` 用于安全卸载（关闭）当前后端镜像。  
  此函数只在程序退出或切换镜像时调用一次，保证 FILE* 生命周期正确。

- 新增 `tf_get_backend(void)`，让其它接口（如底层 I/O、扇区读写）统一获取当前镜像文件指针。

### 2. 扇区读写函数修正

- `read_sector` 和 `write_sector` 函数均改为只使用全局文件指针 `g_backend`，不再每次打开/关闭文件。
- 删除了原实现中的 `fclose(fp);`，仅在附着/分离时关闭一次，避免不当释放导致堆损坏或 double free。
- 读/写操作只包括 `fseek` + `fread` / `fseek` + `fwrite`，确保高效安全的数据访问。
- 每次操作前均检查 `g_backend` 是否已挂载，避免野指针问题。

  例如：

  ```c
  int read_sector(uint8_t *data, uint32_t sector) {
      FILE *fp = g_backend;
      if (!fp) return -1;
      fseek(fp, sector*512, SEEK_SET);
      fread(data, 1, 512, fp);
      return 0;
  }
  ```

### 3. 调试输出辅助

- 在挂载、分离及读写函数加了 [DBG] 打印，便于开发期间随时跟踪资源状态及指针生命周期。

## 修复了什么缺点

### 1. 修复了多次打开/关闭导致的 double free 和堆损坏

- 原实现在 `read_sector`/`write_sector` 里，每次读写都 `fclose(fp);`，会反复关闭同一个 FILE*。  
  这样 FILE* 的内存被系统释放，后续再用（如 seek/read/write）会堆损坏导致 "free(): invalid pointer" 错误，甚至崩溃或数据丢失。
- 现在只在挂载/卸载时管理 FILE*，扇区读写过程始终安全不会破坏内存。

### 2. 统一和简化了后端文件资源管理流程

- 保证了 FILE* 的唯一性和生命周期，不会资源泄漏、野指针访问、越界写等。
- 所有数据访问都在同一个打开文件句柄下完成，提高效率且便于扩展缓存优化。

### 3. 程序稳定性大幅提升

- 所有核心测试功能通过，run 结果无 core dump，无内存非法释放。
- 可安全扩展更多高级功能（如多文件、目录、分区管理等），无需担心基础资源管理问题。

## 为什么这样改动

- **保证 FILE 资源安全、高效、唯一管理** ：只允许 attach/detach 期间关闭文件，不允许在读写过程中反复释放，完全符合标准设计；
- **避免多次释放/内存堆破坏**：C 标准库 FILE* 的释放必须谨慎，否则会导致严重程序崩溃；
- **提高接口通用性和可维护性**：后续所有高层 API（tf_fopen, tf_fclose, tf_fread, tf_fwrite等）都可以稳定依赖统一镜像指针；
- **更清晰调试和日志**：便于开发和故障排查。

---

本次改动彻底消除了因为错误 fclose 导致的所有核心稳定性问题，为 FAT32 文件系统后期开发打下了坚实基础。