---
title: linux pipe 筆記(一)
date: 2020-10-20 22:56:28
tags: Linux
---
## 前言
以下紀錄近期 trace linux kernel module `pipe` 的一些筆記與心得。
## 緣起
有朋友在進行關於 `pipe` 的實驗，原始碼如下:
```c=
int main(){
    char *arg1[] = {"ls", NULL};
    char *arg2[] = {"cat","-n", NULL};
    int p[2];
    if(pipe(p))
        return 0;

    for(int i = 0; i < 2; i++){
        pid_t pid;
        pid = fork();
        if(pid == 0){
            if(i == 0){
                close(p[0]);
                dup2(p[1], 1);
                execvp(arg1[0], arg1);
                close(p[1]);
                exit(0);
            }else if(i == 1){
                close(p[1]);
                dup2(p[0], 0);

                execvp(arg2[0], arg2);
                close(p[0]);
                exit(0);
            }
        }else{
            int status;
            waitpid(pid, &status, 0);
        }
    }
}
```

預期的功能如下：
```bash
$ ls | cat -n
```
預期輸出(該資料夾中有 `main.c` 與 `main` 兩個檔案的狀況下):
```bash
$
     1	main
     2	main.c
$
```

可是卻發生 `cat` 無法終止的問題 (輸出完預期的字串後，一直停留在 `cat` 中不會停止)。
實際輸出:
```bash
$
     1	main
     2	main.c
```

首先，來了解上述實作涉及的函式。
## System calls
### dup2
```c
int dup2(int oldfd, int newfd);
```
**System call** `dup` 會複製給定的 `file descriptor` 並返回複製後的 `file descriptor` 。而 `dup2` 則可以指定要複製的 `file descriptor` 。換句話說 `dup` 系列的指令可以複製並指定特定 `file descriptor` 所指向的 `file description` 。關於 `file descriptor` 與 `file description` 在這篇[文章](https://kkc.github.io/2020/08/22/file-descriptor/) 中有詳細的說明。

### pipe
```c
int pipe(int pipefd[2]);
```
**System call** `pipe` 會開啟一個可以跨行程( process )溝通的 `FIFO buffer` ( `data channel` ) ，並返回該 `channel` 的兩端(讀端與寫端)。資料流向為 `pipefd[1]` -> `pipefd[0]` 。經常用法為: 先開啟 `pipe` 後透過 `fork` 開啟子行程，這樣就可以透過先前開啟的 `pipe` 進行跨行程溝通。

---
而上述實作的目標便是透過以上兩個 `system call` 將父行程的輸出導向子行程的輸入。

## cat 的終止條件
`cat` 是在 `linux` 上經常使用的工具,一直以來都將 `cat` 作為將檔案輸出至終端機的工具。然而我竟然不知道其實際運作原理，著實慚愧。
參閱 [man page](https://man7.org/linux/man-pages/man1/cat.1.html) 說明。 `cat` 其實是 `concatenate` (聯集)的縮寫，其主要功能是將給定的檔案聯集並輸出至 `STDOUT` 。而當沒有指定檔案時，則會聆聽 `STDIN` 作為輸入。
對應實作的程式碼如下:
(此章節中若未特別提及，原始碼皆來自 [cat.c](https://github.com/coreutils/coreutils/blob/master/src/cat.c))
```c=
if (STREQ (infile, "-"))
{
  have_read_stdin = true;
  input_desc = STDIN_FILENO;
  ...
}
else
{
  input_desc = open (infile, file_open_mode);
  ...
}
```

透過以上實作 `cat` 會依序開啟參數中所給定的每個檔案。
之後透過呼叫函數
```c
static bool cat(...)
```
來讀取開啟檔案的內容。

在函數 `cat` 中會透過
```c
n_read = safe_read (input_desc, inbuf, insize);
```
來讀取資料。其中 `safe_read` 是對於 `read` 的包裝，詳細內容請參考[原始碼](https://github.com/coreutils/gnulib/blob/master/lib/safe-read.c)，其中有對於 `MACRO` 的精妙應用。

由於 `cat` 主要是作為讀取檔案的工具，因此可以推斷函數 `cat` 的離開條件應為讀取至檔案結尾 (EOF) ，在 `safe_read` 的原始碼中可以看到以下註解。
```
Return the actual number of bytes read(written), 
zero for EOF
```
另外於 `read` 的 `man page` 中也可以看到以下敘述。
```
On success, the number of bytes read is returned (zero indicates end of file)
```
因此推測函數 `cat` 會將 `n_read == 0` 作為離開的條件。
果不其然可以由原始碼中看到
```c=
if (n_read == 0)
{
  write_pending (outbuf, &bpout);
  newlines2 = newlines;
  return true;
}
```

到此可知，最後我們需要讓 `pipe` 的 `read` 回傳零以結束 `cat` 。

待續。
## Reference
[dup man page](https://man7.org/linux/man-pages/man2/dup.2.html)
[Linux 的 file descriptor 筆記](https://kkc.github.io/2020/08/22/file-descriptor/)
[pipe man page](https://man7.org/linux/man-pages/man2/pipe.2.html)
[cat man page](https://man7.org/linux/man-pages/man1/cat.1.html)
[cat source code](https://github.com/coreutils/coreutils/blob/master/src/cat.c)
[safe_read source code](https://github.com/coreutils/gnulib/blob/master/lib/safe-read.c)
