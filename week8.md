## File system

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#define SMAX 128

int main() {
  close(0);                      // 關閉標準輸入 stdin
  close(1);                      // 關閉標準輸出 stdout
  int a = open("a.txt", O_RDWR);         // 此時 open，會找沒被使用的最小檔案代號 0
  int b = open("b.txt", O_CREAT|O_RDWR, 0644); // 此時 open，會找沒被使用的最小檔案代號 1
  char line[SMAX];
  gets(line);                    // 從 0 (a.txt) 讀入一行字 line
  puts(line);                    // 輸出 line 到 1 (b.txt)
  printf("a=%d, b=%d\n", a, b);
}

```

![image-20220504142207703](C:\Users\z22756392z\AppData\Roaming\Typora\typora-user-images\image-20220504142207703.png)

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#define SMAX 128

int main() {
  int fda = open("a.txt", O_RDWR);          // 打開檔案 a.txt 並取得代號 fda
  int fdb = open("b.txt", O_CREAT|O_RDWR, 0644);  // 打開檔案 b.txt 並取得代號 fdb
  dup2(fda, 0);                             // 複製 fda 到 0 (stdin)
  dup2(fdb, 1);                             // 複製 fdb 到 1 (stdout)
  char line[SMAX];
  gets(line);                               // 從 0 (a.txt) 讀入一行字 line
  puts(line);                               // 輸出 line 到 1 (b.txt)
}

```

![image-20220504142652430](C:\Users\z22756392z\AppData\Roaming\Typora\typora-user-images\image-20220504142652430.png)

#### **application : log**

```
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
  int fdb = open("log.txt", O_APPEND|O_CREATE|O_RDWR, 0644);  // 打開檔案 log.txt 並取得代號 fdb
  dup2(fdb, 2); // 複製 fdb 檔案描述子到 2 (stderr)
  fprintf(stderr, "Warning: xxx\n");
  fprintf(stderr, "Error: yyy\n");
}

```



## shell

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#define SMAX 256

// 將檔案讀入成為字串
int readText(char *file, char *text, int size) {
  FILE *f = fopen(file, "r");
  int n = fread(text, 1, size, f);
  fclose(f);
  return n;
}

int main(int argc, char *argv[]) {
  char ipath[SMAX], path[SMAX], cmd[SMAX], fullcmd[SMAX], pathFile[SMAX];
  getcwd(ipath, SMAX-1); // 取得初始路徑
  strcpy(path, ipath);   // path = ipath
  sprintf(pathFile, "%s/path.txt", ipath); // pathFile=<ipath>/path.txt
  while (1) { // 不斷等待使用者輸入命令並執行之
    printf("myshell:%s $ ", path); // 顯示提示訊息
    fgets(cmd, SMAX-1, stdin);                     // 等待使用者輸入命令
    strtok(cmd, "\n");             // 切掉 \n
    if (strcmp(cmd, "exit")==0) break;
    sprintf(fullcmd, "cd %s;%s;pwd>%s", path, cmd, pathFile); // fullcmd = 切到 path; 使用者輸入的命令; 將路徑存入 pathFile 中。
    system(fullcmd);               // 執行 fullcmd 
    readText(pathFile, path, SMAX);// 讀 pathFile 檔取得路徑
    strtok(path, "\n");            // 切掉 \n
  }
}

```





## net



```c
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <arpa/inet.h>
#include <time.h>
#include <assert.h>
#include <sys/wait.h>

#define PORT 8080

int main(int argc, char *argv[]) {
  struct sockaddr_in serv_addr;
  memset(&serv_addr, 0, sizeof(serv_addr));
  serv_addr.sin_family = AF_INET;
  serv_addr.sin_port = htons(PORT);
  assert(inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr) > 0);

  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  assert(sockfd >=0);
  assert(connect(sockfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) >= 0);
  char recvBuff[1024];
  int n;
  while ((n = read(sockfd, recvBuff, sizeof(recvBuff)-1)) > 0) {
    recvBuff[n] = 0;
    fputs(recvBuff, stdout);
  }
  return 0;
}

```



```c
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <arpa/inet.h>
#include <time.h>
#include <assert.h>
#include <sys/wait.h>

#define PORT 8080

int main(int argc, char *argv[]) {
  int listenfd = socket(AF_INET, SOCK_STREAM, 0);
  assert(listenfd >= 0);
  struct sockaddr_in serv_addr;
  memset(&serv_addr, 0, sizeof(serv_addr));
  serv_addr.sin_family = AF_INET;
  serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
  serv_addr.sin_port = htons(PORT);

  assert(bind(listenfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) >= 0);
  assert(listen(listenfd, 10) >= 0); // 最多十個同時連線
  char sendBuff[1025];
  memset(sendBuff, 0, sizeof(sendBuff));
  while(1) {
    int connfd = accept(listenfd, (struct sockaddr*)NULL, NULL);
    assert(connfd >= 0);
    time_t ticks = time(NULL);
    snprintf(sendBuff, sizeof(sendBuff), "%.24s\r\n", ctime(&ticks));
    assert(write(connfd, sendBuff, strlen(sendBuff)) >=0);
    close(connfd);
    sleep(1);
  }
}

```

