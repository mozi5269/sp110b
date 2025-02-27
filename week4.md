# 程式碼解說 03-compiler(續)



> exp0var.c: 把指令轉換成組合語言，一樣只能處裡單一字符，把結果變成hack CPU的組合語言，可以記憶變數(x+y)，到這裡還是只能判斷 +-

```c
#include <stdio.h>
#include <assert.h>
#include <string.h>
#include <ctype.h>

int tokenIdx = 0;
char *tokens; // 紀錄容器位置(argv[1])

int E();
int F();

void error(char *msg) {
  printf("%s", msg);
  assert(0);  // 只要值是0或是false就強制退出，並印出stack
}

char ch() {  // 取得字符並傳回
  char c = tokens[tokenIdx];
  return c;
}

char next() {  // 前往下一個字符
  char c = ch();
  tokenIdx++;
  return c;
}

int isNext(char * set) {  // 判斷下一個自符又沒有在集合裡面
  char c = ch(); 
  return (c!='\0' && strchr(set, c)!=NULL);
}

int nextTemp() {  // 創造新的臨時變數
  static int tempIdx = 0;
  return tempIdx++;
}

void genOp1(int i, char c) {
  printf("# t%d=%c\n", i, c);
  // t1=3 轉成 @3; D=A; @t1; M=D
  // t1=x 轉成 @x; D=M; @t1; M=D
  printf("@%c\n", c);
  char AM = (isdigit(c)) ? 'A':'M';
  printf("D=%c\n", AM);
  printf("@t%d\n", i);
  printf("M=D\n");
}

void genOp2(int i, int i1, char op, int i2) {
  printf("# t%d=t%d%ct%d\n", i, i1, op, i2);
  // t0=t1+t2 轉成 @t1; D=M; @t2; D=D+M; @t0; M=D;
  printf("@t%d\n", i1);
  printf("D=M\n");
  printf("@t%d\n", i2);
  printf("D=D%cM\n", op);
  printf("@t%d\n", i);
  printf("M=D\n");
}

// F =  Number | ID | '(' E ')'
int F() {
  int f;
  char c = ch();
  if (isdigit(c) || (c>='a'&&c<='z')) {  // 這個可以處裡變數
    next(); // skip c
    f = nextTemp();
    genOp1(f, c);
  } else if (c=='(') { // '(' E ')'
    next();
    f = E();
    assert(ch()==')');  // 誇號開頭就要誇號結束，如果沒有誇號就強制退出
    next();
  } else {
    error("F = (E) | Number fail!");
  }
  return f; 
}

// E = F ([+-] F)*
int E() {
  int i1 = F();
  while (isNext("+-")) {  // 偵測到 +- 就前往下一個字符
    char op=next();  // 前往下一個字符
    int i2 = F();  
    int i = nextTemp();
    genOp2(i, i1, op, i2);
    i1 = i;
  }
  return i1;
}

void parse(char *str) {
  tokens = str;
  E();
}

int main(int argc, char * argv[]) {
  printf("=== EBNF Grammar =====\n");
  printf("E=F ([+-] F)*\n");
  printf("F=Number | ID | '(' E ')'\n");
  printf("==== parse:%s ========\n", argv[1]);
  parse(argv[1]);
}

```



## lexer

> lexer.c : 詞彙掃描程式，可以分辨字串(" ")，和字元，可以讀檔後對檔案做分析

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>

#define TMAX 10000000  
#define SMAX 100000

enum { Id, Int, Keyword, Literal, Char };

char *typeName[5] = {"Id", "Int", "Keyword", "Literal", "Char"};

char code[TMAX];
char strTable[TMAX], *strTableEnd=strTable; // 創建一個字符列表，後面的pointer代表迭代位置
char *tokens[TMAX];
int tokenTop=0;
int types[TMAX];

// define 函數會加上誇號()，函數的值會直接return到前面
#define isDigit(ch) ((ch) >= '0' && (ch) <='9')  

#define isAlpha(ch) (((ch) >= 'a' && (ch) <='z') || ((ch) >= 'A' && (ch) <= 'Z'))

int readText(char *fileName, char *text, int size) {  // 使用C語言讀檔案
  FILE *file = fopen(fileName, "r");
  int len = fread(text, 1, size, file);
  text[len] = '\0';
  fclose(file);
  return len;
}

/* strTable =
#\0include\0"sum.h"\0int\0main\0.....
*/
char *next(char *p) {
  while (isspace(*p)) p++;

  char *start = p; //         include "sum.h"
                   //         ^      ^
                   //  start= p      p
  int type;
  if (*p == '\0') return NULL;
  if (*p == '"') {
    p++;
    while (*p != '"') p++;
    p++;
    type = Literal;
  } else if (*p >='0' && *p <='9') { // 數字
    while (*p >='0' && *p <='9') p++;
    type = Int;
  } else if (isAlpha(*p) || *p == '_') { // 變數名稱或關鍵字
    while (isAlpha(*p) || isDigit(*p) || *p == '_') p++;
    type = Id;
  } else { // 單一字元
    p++;
    type = Char;
  }
  int len = p-start;
  char *token = strTableEnd;  // token紀錄開始位置
  // strncpy 可以把start的位置往後len存到前面的 starTableEnd(pointer) 裡面
  strncpy(strTableEnd, start, len);  
  strTableEnd[len] = '\0';  // printf遇到這個會自動停止
  strTableEnd += (len+1);  // 迭代到下一個位置
  types[tokenTop] = type;
  tokens[tokenTop++] = token;
  printf("token=%s\n", token);
  return p;
}

void lex(char *code) {  // 程式主要運行的地方
  char *p = code;
  while (1) {
    p = next(p);
    if (p == NULL) break;
  }
}

void dump(char *strTable[], int top) {
  for (int i=0; i<top; i++) {
    printf("%d:%s\n", i, strTable[i]);
  }
}

int main(int argc, char * argv[]) {
  readText(argv[1], code, sizeof(code)-1);
  puts(code);  // 打印一次，看到程式長怎樣
  lex(code);
  dump(tokens, tokenTop);
}
```



## 03a-compiler

使用CodeBlock的`mingw32-make`，可以編譯Makefile，執行裡面的命令!!

> compiler.c : 比較完整的compiler

```c
#include <assert.h>
#include "compiler.h"

int E();
void STMT();
void IF();
void BLOCK();

int tempIdx = 0, labelIdx = 0;

#define nextTemp() (tempIdx++)
#define nextLabel() (labelIdx++)
#define emit printf

int isNext(char *set) {  // 判斷下一個字符是否符合條件
  char eset[SMAX], etoken[SMAX];
  // 把set放入eset裡面，可以照著中間字串的形式 elif  el if
  // 加上空格是因為確認要比對文字，通常是用雜湊表去比對，不過代碼就會更長了
  sprintf(eset, " %s ", set);  
  sprintf(etoken, " %s ", tokens[tokenIdx]);
  return (tokenIdx < tokenTop && strstr(eset, etoken) != NULL);
}

int isEnd() {
  return tokenIdx >= tokenTop;
}

char *next() {
  // printf("token[%d]=%s\n", tokenIdx, tokens[tokenIdx]);
  return tokens[tokenIdx++];
}

char *skip(char *set) {
  if (isNext(set)) {
    return next();
  } else {
    printf("skip(%s) got %s fail!\n", set, next());
    assert(0);
  }
}

// F = (E) | Number | Id
int F() {
  int f;
  if (isNext("(")) { // '(' E ')'
    next(); // (
    f = E();
    next(); // )
  } else { // Number | Id
    f = nextTemp();
    char *item = next();
    emit("t%d = %s\n", f, item);
  }
  return f;
}

// E = F (op E)*
int E() {
  int i1 = F();
  while (isNext("+ - * / & | ! < > =")) {
    char *op = next();
    int i2 = E();
    int i = nextTemp();
    emit("t%d = t%d %s t%d\n", i, i1, op, i2);
    i1 = i;
  }
  return i1;
}

// ASSIGN = id '=' E;   // 普通陳述句
void ASSIGN() {
  char *id = next();
  skip("=");  // 有點像cin.get()，會直接吃掉，沒有找到就會報錯
  int e = E();
  skip(";");
  emit("%s = t%d\n", id, e);
}

// WHILE = while (E) STMT
void WHILE() {  // 遞迴下降法，這邊只是打印出邏輯，沒有真的迴圈
  int whileBegin = nextLabel();  // 讓compiler可以用if goto 去跳
  int whileEnd = nextLabel();
  emit("(L%d)\n", whileBegin);  // 跟printf一樣
  skip("while");
  skip("(");
  int e = E();
  emit("if not T%d goto L%d\n", e, whileEnd);
  skip(")");
  STMT();
  emit("goto L%d\n", whileBegin);
  emit("(L%d)\n", whileEnd);
}

// STMT = WHILE | BLOCK | ASSIGN  // 主要判斷邏輯
void STMT() {
  if (isNext("while"))
    return WHILE();
  // else if (isNext("if"))
  //   IF();
  else if (isNext("{"))
    BLOCK();
  else 
    ASSIGN();
}

// STMTS = STMT*
void STMTS() {
  while (!isEnd() && !isNext("}")) {  // 碰到檔案結束和碰到右大誇號就結束
    STMT();
  }
}

// BLOCK = { STMTS }
void BLOCK() {
  skip("{"); 
  STMTS();
  skip("}");
}

// PROG = STMTS
void PROG() {
  STMTS();
}

void parse() {
  printf("============ parse =============\n");
  tokenIdx = 0;
  PROG();
}
```






