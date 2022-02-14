---
title: linux pipe 筆記(二)
date: 2020-10-22 22:41:16
tags: Linux
---

## pipe 原理
以下 `pipe` 原始碼來自 `linux v5.4` ， [bootlin](https://elixir.bootlin.com/linux/v5.4/source/fs/pipe.c) 超連結。

### 開啟
開啟新的 `pipe` 不是透過一般的 `open` ，而是使用獨立的 **system call** `pipe` 。該 **system call** 實作如下。
```c=
SYSCALL_DEFINE1(pipe, int __user *, fildes)
{
	return do_pipe2(fildes, 0);
}
```

該 **system call** 的主要實作為函數 `do_pipe2()` 。在該函數中會
1. 建立新的 `pipe` 物件。
2. 獲取兩個 `file descriptor` 並將其回傳至 `user space` 。
4. 將兩個 `file descriptor` 指向對應的 `file description` 。
透過上述步驟就可以將原本作為參數傳入的陣列填上對應的兩個 `file descriptor` 了。

在 `do_pip2e()` 中，透過 `__do_pipe_flags()` 呼叫 `create_pipe_files()` 建立新的 `pipe` 物件。
在 `create_pipe_files()` 中呼叫 `get_pipe_inode()` 。在 `get_pipe_inode()` 函數中，可以看到其創建一個 `struct pipe_inode_info` 的物件並透過 `inode` 物件的 `i_pipe` 成員指向他。
檢視 `struct pipe_inode_info` 內容，[原始碼](https://elixir.bootlin.com/linux/v5.4/source/include/linux/pipe_fs_i.h#L48)，以下節錄我們感興趣的部份：
```c=
struct pipe_inode_info {
    ...
	unsigned int readers;
	unsigned int writers;
    ...
	struct pipe_buffer *bufs;
    ...
};
```
可以發現，在 `struct pipe_inode_info` 中包含了兩個計數器。根據變數名稱可以推測分別儲存此物件當前**讀者**與**寫者**的數量。

可以在 `get_pipe_inode()` 中看到，這兩個計數器如預期般的被初始化為 `1` 。
```c=
pipe->readers = pipe->writers = 1;
```
另外在該函數中
```c=
inode->i_pipe = pipe;
```
透過上方程式碼由指標將新配置的 `pipe` 物件變成 `inode` 的成員。

在 `get_pipe_inode()` 回傳新建立的 `inode` 物件後
```c=
f = alloc_file_pseudo(inode, pipe_mnt, "",
				O_WRONLY | (flags & (O_NONBLOCK | O_DIRECT)),
				&pipefifo_fops);
...
f->private_data = inode->i_pipe;
res[0] = alloc_file_clone(f, O_RDONLY | (flags & O_NONBLOCK),
				  &pipefifo_fops);
...
res[0]->private_data = inode->i_pipe;
res[1] = f;
```
透過以上程式碼將 `pipe_inode_info` 物件變為兩個 `file description` 的 `private_data` 成員。而這兩個 `file description` 內容大致相同(由於是透過 `clone` 複製)，差別只在於讀寫權限。由於回傳的 `pipefd[1]` 為寫端而 `pipefd[0]` 為讀端，在此可以發現最初開啟的 `f` 權限為可讀寫，而 `f` 也被存放至 `res[1]` 。而之後 `clone` 的權限則為唯讀，並被存放至 `res[0]` 。以上的行為皆與 `pipe man page` 的敘述相符。

而 `res` 所指向的開啟的檔案，在回到 `do_pipe_flags()` 中後，透過
```c=
/* files 即是 res */
fd_install(fd[0], files[0]);
fd_install(fd[1], files[1]);
```
安插至個別的 `file descriptor` 上了。所以實際上 `pipe` 所開啟的兩個 `file descriptor` 指向的都是有相同內容的兩個 `file` 物件。如上所述，兩物件差異只在於讀寫權限。而該兩物件的 `private data` 則都指向同一 `pipe_inode_info` 物件。

至此， `pipe` 的開起與其結構大致告一段落。
接下來將探討 `pipe` 的讀寫機制並修復原有程式的錯誤。

待續。
## Reference
[Linux 的 file descriptor 筆記](https://kkc.github.io/2020/08/22/file-descriptor/)
[pipe man page](https://man7.org/linux/man-pages/man2/pipe.2.html)
[bootlin pipe.c](https://elixir.bootlin.com/linux/v5.4/source/fs/pipe.c)
